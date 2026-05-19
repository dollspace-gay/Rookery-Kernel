# Tier-3: drivers/firewire/{core-card,core-cdev,core-transaction,core-iso}.c — IEEE 1394 core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/firewire/core-card.c
  - drivers/firewire/core-cdev.c
  - drivers/firewire/core-device.c
  - drivers/firewire/core-iso.c
  - drivers/firewire/core-topology.c
  - drivers/firewire/core-transaction.c
  - drivers/firewire/core.h
  - drivers/firewire/ohci.c
  - drivers/firewire/ohci.h
  - drivers/firewire/sbp2.c
  - drivers/firewire/net.c
  - drivers/firewire/nosy.c
  - include/linux/firewire.h
  - include/uapi/linux/firewire-cdev.h
-->

## Summary

The `firewire` subsystem is the IEEE 1394 (a.k.a. FireWire, i.LINK) stack: it presents an isochronous + asynchronous packet bus with up to 63 nodes per local bus and supports DMA-capable peer nodes that can directly read and write *physical* memory on the host via the IEEE 1394 PHY-layer "Physical Response" mechanism. The core consists of `core-card.c` (per-card `fw_card` lifecycle + bus reset), `core-device.c` + `core-topology.c` (self-ID parsing, per-node `fw_device` enumeration), `core-transaction.c` (async read/write/lock transactions, request/response routing, retry timer wheel), `core-iso.c` (iso context lifecycle, packet queue), `core-cdev.c` (the `/dev/fw%u` userspace chardev — direct iso, async, FCP), and `ohci.c` (the OHCI-1394 PCI HBA driver, by far the dominant backend, plus `nosy.c` for the dedicated bus-analyzer card). Sub-clients include `sbp2` (storage), `net` (IPv4/IPv6 over 1394, RFC 2734/3146), and a `nosy-user` ABI for raw bus tracing.

Because IEEE 1394 PHY accepts unsolicited "physical-write" packets that the OHCI hardware fulfils *without CPU mediation*, every FireWire host is a peer-DMA bus identical in risk profile to Thunderbolt PCIe tunnelling; full IOMMU mediation is mandatory on any platform exposing a FireWire port externally.

Sources: `core-card.c` (~837 lines), `core-cdev.c` (~1911 lines), `core-transaction.c` (~1508 lines), `core-iso.c` (~459 lines), `core-device.c`, `core-topology.c`, `ohci.c`, `sbp2.c`, `net.c`, `nosy.c`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct fw_card` (`include/linux/firewire.h`) | per-HBA card | `drivers::firewire::Card` |
| `struct fw_card_driver` | per-HBA backend ops (`enable`, `read_phy_reg`, `update_phy_reg`, `send_request`, `send_response`, `cancel_packet`, `enable_phys_dma`, `read_csr`, `write_csr`, `allocate_iso_context`, `send_iso`, `start_iso`, `stop_iso`, `queue_iso`, `flush_iso_completions`) | `drivers::firewire::CardDriver` |
| `struct fw_device` | per-node remote-device proxy | `drivers::firewire::FwDevice` |
| `struct fw_transaction` (`include/linux/firewire.h`) | per-pending async transaction | `drivers::firewire::Transaction` |
| `struct fw_request` | inbound request awaiting application response | `drivers::firewire::FwRequest` |
| `struct fw_address_handler` | per-CSR-range handler for inbound async | `drivers::firewire::AddressHandler` |
| `struct fw_iso_context` / `fw_iso_buffer` | iso context + DMA-backed packet buffer | `drivers::firewire::IsoContext` / `IsoBuffer` |
| `fw_card_initialize(card, driver, dev)` / `fw_card_add(card, max_speed, hw_guid)` / `fw_core_remove_card(card)` / `fw_card_release(kref)` / `fw_card_read_cycle_time(card, &t)` | per-HBA lifecycle + cycle-time read | `Card::initialize` / `_add` / `_remove` / `_release` / `_read_cycle_time` |
| `fw_schedule_bus_reset(card, delayed, short_reset)` | trigger PHY bus reset | `Card::schedule_bus_reset` |
| `__fw_send_request(card, t, tcode, dest_id, gen, speed, off, payload, len, callback, ts_cb, data)` / `fw_send_request(...)` / `fw_send_request_with_tstamp(...)` / `fw_run_transaction(...)` | async transaction submit + sync wrapper | `Transaction::send` / `_run` |
| `fw_cancel_transaction(card, t)` | cancel in-flight transaction (timeout / shutdown) | `Transaction::cancel` |
| `fw_core_handle_request(card, packet)` / `fw_core_handle_response(card, packet)` | OHCI calls these on per-packet IRQ | `Card::handle_request` / `_handle_response` |
| `fw_core_add_address_handler(handler, region)` / `fw_core_remove_address_handler(handler)` | CSR range registration | `AddressHandler::add` / `_remove` |
| `fw_send_response(card, request, rcode)` / `fw_fill_response(response, request_header, rcode, payload, len)` / `fw_get_request_speed(request)` / `fw_request_get_timestamp(request)` / `fw_rcode_string(rcode)` | per-request response + helpers | `FwRequest::*` |
| `fw_core_add_descriptor(desc)` / `fw_core_remove_descriptor(desc)` | local-node config-ROM descriptor add | `Subsystem::*_descriptor` |
| `fw_high_memory_region` (>1MB physical-DMA region) | the "high memory" CSR space exposed to peers when PHY-DMA enabled | `Subsystem::high_memory_region` |
| `fw_iso_buffer_init(buffer, card, page_count, direction)` / `_destroy(buffer, card)` | iso DMA-buffer alloc | `IsoBuffer::init` / `_destroy` |
| `__fw_iso_context_create(card, type, channel, speed, header_size, callback, data)` / `fw_iso_context_destroy(ctx)` / `_start(ctx, cycle, sync, tags)` / `_stop(ctx)` / `_queue(ctx, pkt, buffer, offset)` / `_queue_flush(ctx)` / `_flush_completions(ctx)` | iso context lifecycle | `IsoContext::*` |
| `fw_iso_resource_manage(card, gen, channels_mask, &bw_used, &chs_used, allocate)` | iso bandwidth + channel arbitration via IRM | `Subsystem::iso_resource_manage` |

## Compatibility contract

REQ-1: Per-card `fw_card_driver` exposes the OHCI-equivalent primitives so any 1394 HBA (in practice always `ohci.c`) can plug in. PHY register read/write, AT/AR async transmit/receive, IT/IR iso transmit/receive, IRM (Isochronous Resource Manager) responsibilities, cycle master, bus reset.

REQ-2: IEEE 1394 self-ID round on every bus reset: PHY arbitrates, every node emits its self-ID packet, `core-topology.c` builds per-card `fw_card.topology` tree, allocates per-node IDs, walks each node's Config ROM to populate `fw_device` children.

REQ-3: Async transactions: per `fw_transaction` carries (tcode, destination_id, generation, speed, offset, payload). On send, OHCI queues into AT context FIFO; on response, OHCI calls `fw_core_handle_response` with rcode + payload. Per-transaction retry handled by hardware up to `max_retries`.

REQ-4: Inbound request dispatch: per CSR-range `fw_address_handler` registered by clients (sbp2, FCP, application via cdev). Per `core-transaction.c` lookup, dispatch to handler's `address_callback`. Handler issues `fw_send_response`.

REQ-5: Iso contexts: per-card up to OHCI-supported count (typically 4 IR + 4 IT contexts). Per-context buffer is page-vector mapped DMA. `_queue(packet, buffer, offset)` enqueues descriptor; OHCI fires per-packet or per-buffer-completion IRQ; callback receives header + buffer offset.

REQ-6: PHY-layer "Physical DMA" path: when enabled (`fw_card_driver.enable_phys_dma(card, node_id, generation)`), OHCI accepts unsolicited 1394 async-read/write packets to the "Physical Response Unit" address range and the HBA returns / writes physical RAM *without CPU interrupt*. This is the high-risk surface that mandates IOMMU + LOCKDOWN.

REQ-7: Userspace ABI (`/dev/fw%u` via `core-cdev.c`) versioned via `FW_CDEV_VERSION` (currently 6) with `FW_CDEV_IOC_GET_INFO` returning the version + bus reset gen + local node id. Per-fd transactions tracked separately from kernel clients; resource teardown on close drains all pending iso contexts and async transactions.

REQ-8: Function Control Protocol (FCP): per-client registers handlers for FCP_REQUEST (`0xfffff0000b00`) and FCP_RESPONSE (`0xfffff0000d00`) address ranges with the IEEE 1394 12-bit length cap (max 512 bytes payload per FCP frame).

REQ-9: Bus reset: 1394 SHORT_RESET (~0.1 ms, preserves node IDs) or LONG_RESET (~166 ms, full re-enumeration). Per `fw_schedule_bus_reset(card, delayed, short)`, optionally delayed via workqueue. On reset complete, `core-topology` builds new tree, increments `card.generation`, expires all in-flight transactions whose `t.generation != card.generation` with `RCODE_GENERATION` (-100).

REQ-10: Per-card workqueue + timer wheel: `fw_card.work` handles bus reset + topology rebuild + device probe; `card.split_timeout_*` timers track async-transaction split-response timeouts (default 800 ms).

REQ-11: IRM (Isochronous Resource Manager) protocol: per-bus elected node holds CSR_BANDWIDTH_AVAILABLE + CSR_CHANNELS_AVAILABLE; `fw_iso_resource_manage` arbitrates per-channel allocation via lock transactions to IRM.

REQ-12: Config ROM: per-card local ROM holds `fw_descriptor`-chain entries registered by `fw_core_add_descriptor`; remote nodes' Config ROM read out at bus-reset enumeration into `fw_device.config_rom`.

## Acceptance Criteria

- [ ] AC-1: Modprobe `firewire-ohci` on a system with an OHCI-1394 card; `/sys/class/firewire/` populates `fw0` host node + any attached peer nodes.
- [ ] AC-2: Modprobe `firewire-sbp2`; an attached FireWire HDD enumerates as `/dev/sd*` via SCSI.
- [ ] AC-3: Bus reset (unplug and replug an attached device) increments `card.generation`, expires pending async tx, re-enumerates.
- [ ] AC-4: Iso: a `libraw1394`-based DV camcorder capture via `/dev/fw0` ioctl receives frames without packet loss at 25 MB/s sustained.
- [ ] AC-5: FCP test: register an FCP_REQUEST handler via cdev `FW_CDEV_IOC_ALLOCATE`; remote device's FCP write delivered with correct payload + 12-bit length.
- [ ] AC-6: PHY-DMA disabled by default: `cat /sys/module/firewire_ohci/parameters/remote_dma` shows `N`; admin must explicitly enable per-bus.
- [ ] AC-7: IOMMU group containment: every `firewire-ohci` PCI function lives in its own IOMMU group when the platform IOMMU is enabled.
- [ ] AC-8: `nosy` bus-analyzer chardev returns raw packet traces from a dedicated nosy-card (CONFIG_FIREWIRE_NOSY=y on supported HW).

## Architecture

`Card` is the per-1394-HBA runtime:

```
struct Card {
  driver: &'static CardDriver, device: Arc<Device>, kref: Kref,
  guid: u64,                                  // EUI-64
  node_id: AtomicU16, generation: AtomicU32,  // bumped per bus reset
  current_tlabel: AtomicU16, tlabel_mask: AtomicU32,
  transaction_list: SpinLock<List<Arc<Transaction>>>,
  split_timeout_hi: u32, split_timeout_lo: u32, split_timeout_cycles: u32, split_timeout_jiffies: u64,
  flush_timer: Timer, link_speed: u8, max_receive: u8,
  topology_map: NonNull<[u32]>, config_rom: NonNull<[u32]>,
  bm_retries: u32, bm_node_id: u32, bm_state: BmState, bm_work: Work, bm_abort_timer: Timer,
  irm_node: AtomicPtr<FwNode>, root_node: AtomicPtr<FwNode>, local_node: AtomicPtr<FwNode>,
  card_status: AtomicU32,                     // BUS_RESET_PENDING | etc.
  broadcast_channel_allocated: bool, broadcast_channel: u32, broadcast_channel_keepalive: DelayedWork,
}

struct Transaction {
  tlabel: u8, is_split_transaction: bool, timestamp: AtomicI64, split_timeout_timer: Timer,
  packet: FwPacket,             // sent header + payload
  callback: Option<fn(card, rcode, payload, len, data)>,
  callback_ts: Option<fn(card, rcode, req_ts, resp_ts, payload, len, data)>,
}

struct IsoContext { card: Arc<Card>, context_type: IsoType, channel: i32, speed: u32, header_size: usize, callback: IsoCallback }

struct AddressHandler { offset: u64, length: u64, address_callback: AddressCb }
```

Per-card init `Card::initialize`:
1. `kref_init(&card.kref)`.
2. Allocate `topology_map`, `config_rom`, transaction-list lock.
3. Hook into `firewire_workqueue` for `bm_work` + topology work.

Per-card add `Card::add(max_speed, hw_guid)`:
1. `card.guid = hw_guid`.
2. Build initial Config ROM with vendor/model/textual entries.
3. `device_register(&card.device)` under firewire class.
4. Driver enables PHY: link up, initial bus reset, first self-ID round populates topology.

Bus-reset path:
1. PHY (or `fw_schedule_bus_reset`) triggers reset.
2. OHCI fires bus-reset IRQ → `card_status = BUS_RESET_PENDING`.
3. Self-ID packets received by OHCI, captured into `topology_map`.
4. `bm_work` runs: parse self-IDs → build tree → elect IRM → expire stale transactions (gen mismatch) → for each new node, alloc `fw_device` + read Config ROM.
5. Per `fw_device` added/removed, drivers (sbp2, net, fwserial) match via Config ROM.

Async send `Transaction::send`:
1. Allocate tlabel from `tlabel_mask`.
2. Build outgoing packet (tcode, dest_id, gen, off, payload).
3. Push onto `card.transaction_list`.
4. Arm `split_timeout_timer` at `card.split_timeout_jiffies`.
5. Call `card.driver.send_request(card, &t.packet)` to push into AT FIFO.
6. On response: OHCI fires AR-resp IRQ → `fw_core_handle_response(card, packet)` → match tlabel → remove from list → invoke `t.callback`.
7. On timeout: `split_timeout_timer` fires → cancel transaction → callback with `RCODE_CANCELLED`.

Inbound request `fw_core_handle_request`:
1. OHCI delivers request packet to `core-transaction.c`.
2. Walk `address_handler_list` for matching CSR offset + length.
3. On match: build `fw_request` (kref'd) + invoke `handler.address_callback`.
4. Handler may complete sync (call `fw_send_response`) or hold the `fw_request` for async completion.

Iso queue `IsoContext::queue(packet, buffer, offset)`:
1. Validate `buffer + offset + packet.payload_length` ≤ buffer.size.
2. Build OHCI DMA descriptor in iso descriptor program.
3. On completion IRQ: `callback(ctx, cycle, header, ...)` fires.

PHY-DMA enable `card.driver.enable_phys_dma(node_id, generation)`:
1. Refuse if `generation != card.generation` (stale request).
2. Validate node_id is in current topology.
3. Program OHCI Physical Response Unit to accept node_id's incoming async-read/write to `fw_high_memory_region`.
4. This is the LOCKDOWN gate: every `enable_phys_dma` invocation crosses the trust boundary, and SHOULD never be reached on a kernel built with `LOCKDOWN_INTEGRITY_MAX` unless an IOMMU has been validated as the gate.

cdev `/dev/fw%u` per-fd operations:
1. `FW_CDEV_IOC_GET_INFO`: return version + GUID + bus gen.
2. `FW_CDEV_IOC_SEND_REQUEST`: build `Transaction` from user struct + payload-copy from user (bounded by 1394 packet limit, 64 KB for async-stream); submit.
3. `FW_CDEV_IOC_ALLOCATE`: register an `AddressHandler` whose callback queues a `fw_cdev_event_request2` for the fd.
4. `FW_CDEV_IOC_CREATE_ISO_CONTEXT`: alloc `IsoContext` bound to fd; userspace mmaps `fw_iso_buffer`.
5. `FW_CDEV_IOC_QUEUE_ISO`: pass packet descriptor batch; bounded by per-context queue depth.
6. On close: drain all per-fd transactions + iso contexts + handlers; release refs.

## Hardening

- Every `Transaction` and `IsoContext` carries `card.generation`-snapshot; bus-reset between submit and process causes `RCODE_GENERATION` rather than UB.
- AT/AR descriptor rings sized at OHCI init; per-ring index validated against ring size on every IRQ.
- Per-fd cdev tracks per-resource handles in `client.resource_idr`; drained on close.
- `fw_core_add_address_handler` validates `region.start < region.end` and disjointness from already-registered regions.
- `fw_send_response` validates `rcode` is one of the spec-defined values; refuses out-of-band rcodes.
- FCP handler enforces 12-bit (max 4095, used as 512-byte block) payload-length cap per IEC 61883.
- Iso `_queue` validates payload + header size against buffer size before descriptor write.
- PHY-DMA `enable_phys_dma` strictly per (node_id, generation); revoked on bus reset.
- cdev userspace pointer paths use `copy_from_user`/`copy_to_user` with explicit length bounds — no raw pointer follow.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `fw_card`, `fw_device`, `fw_transaction`, `fw_request`, `iso_context`, `address_handler`, and the cdev event-queue buffer; copy-in/out paths in `core-cdev.c` use bounded `copy_from_user`/`copy_to_user` only.
- **PAX_KERNEXEC** — firewire core (`core-card.c`, `core-cdev.c`, `core-device.c`, `core-iso.c`, `core-topology.c`, `core-transaction.c`) in W^X kernel text; `fw_card_driver`, `fw_device.config_rom_retrieved`, `address_handler.address_callback`, `iso_context.callback`, `cdev_file_operations` placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel stack on OHCI IRQ entry, `fw_core_handle_request`/`_response`, `bm_work`, `fw_iso_context_queue`, cdev `ioctl`/`mmap` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `fw_card.kref`, `fw_device` lifetime, `fw_request` refs, `fw_transaction` refs; overflow trap defeats bus-reset-race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `fw_request`, async-payload buffers, iso buffer pages on context destroy, and cdev event records so prior-node payloads cannot bleed across hot-swap.
- **PAX_UDEREF** — SMAP/PAN enforced on every `core-cdev.c` ioctl entry (`SEND_REQUEST`, `ALLOCATE`, `QUEUE_ISO`, etc.); userspace pointer deref strictly through canonical helpers.
- **PAX_RAP / kCFI** — `fw_card_driver`, `address_handler.address_callback`, `iso_context.callback`, `fw_descriptor.set`, `fw_transaction.callback`/`_ts`, cdev `file_operations` marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms disclosure of OHCI MMIO bases, `fw_card`/`fw_device` pointers, and EUI-64 GUIDs (in non-CAP_SYSLOG-allowed sysfs paths) behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict bus-reset, self-ID-error, AT/AR overrun, and Config-ROM-parse-fail banners to CAP_SYSLOG.
- **CAP_SYS_RAWIO on /dev/fw*** — chardev open + every `FW_CDEV_IOC_*` ioctl requires CAP_SYS_RAWIO because the cdev exposes raw 1394 async read/write to peer-physical addresses; treat parity with `/dev/mem`.
- **Iso CAP_SYS_ADMIN** — `FW_CDEV_IOC_CREATE_ISO_CONTEXT`, `_QUEUE_ISO`, `_START_ISO`, `_STOP_ISO`, `_ALLOCATE_ISO_RESOURCE` additionally require CAP_SYS_ADMIN; isochronous channels carry real-time DMA bandwidth that can starve other system DMA.
- **FCP 12-bit length cap** — every FCP request/response payload bounded to 512 bytes (12-bit `data_length` field per IEC 61883); refuse OOB.
- **AT/AR ring bounded** — per-OHCI AT request, AT response, AR request, AR response ring sizes set at HBA init; per-descriptor offset validated on every dequeue; ring-overrun triggers controlled HBA reset, not UB.
- **IOMMU mandatory** — firewire-ohci PCI function refused to bind unless platform IOMMU is enabled AND the function lives in its own IOMMU group; `enable_phys_dma` further gated behind LOCKDOWN_INTEGRITY_MAX and a per-bus admin sysfs opt-in (`/sys/bus/firewire/devices/fw%u/remote_dma_allowed`). PHY-layer raw DMA is treated identically to Thunderbolt PCIe tunnelling — full per-device IOMMU domain mandatory.

Rationale: IEEE 1394 PHY-layer Physical Response is *unsolicited peer-DMA without CPU intervention* — exactly the surface that triggered the well-known FireWire forensic-RAM-dump attacks. Any platform exposing a FireWire socket externally is in the same threat model as exposed Thunderbolt. Refusing to bind without an IOMMU, requiring CAP_SYS_RAWIO on `/dev/fw*`, CAP_SYS_ADMIN on iso, LOCKDOWN gating on `enable_phys_dma`, refcount-overflow-trapping on bus-reset-race UAFs, and kCFI on the dense callback web (`fw_card_driver`, `address_handler`, `iso_context.callback`, `cdev` ops) keep the FireWire subsystem from being a back door even when the rest of the system is hardened.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- OHCI-1394 HBA driver (`ohci.c`) — Tier-4.
- `sbp2` storage client — Tier-4.
- `net.c` IPv4/IPv6 over 1394 — Tier-4.
- `nosy.c` bus-analyzer card — Tier-4.
- Implementation code.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `transaction_gen_match` | OOB | every response delivered carries `gen == card.generation`; stale generation forces `RCODE_GENERATION` not UB. |
| `iso_queue_bounded` | OOB | `IsoContext::queue` validates `buffer.offset + packet.payload_length <= buffer.size`. |
| `address_handler_disjoint` | EXCLUSIVITY | new `AddressHandler` rejected if its range overlaps an existing handler. |
| `cdev_resource_no_leak` | UAF | on `/dev/fw*` close, every per-fd `Transaction`, `IsoContext`, and `AddressHandler` is drained before fd release. |

### Layer 2: TLA+

`models/firewire/bus_reset.tla` (parent-declared): proves bus-reset propagation is monotonic — every in-flight transaction either completes before the reset is observed, or completes with `RCODE_GENERATION` after; no transaction silently survives a generation bump.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Transaction::send` post: tlabel allocated from `tlabel_mask`, transaction on `card.transaction_list`, timer armed | `Transaction::send` |
| `fw_core_handle_response` post: matched transaction removed from list, callback invoked exactly once | `Card::handle_response` |
| `enable_phys_dma(node_id, gen)` post: refused unless `gen == card.generation`, `node_id` in current topology, IOMMU group validated | `Card::enable_phys_dma` |
| Bus-reset post: every transaction in `transaction_list` with `t.generation != card.generation` is expired with `RCODE_GENERATION` | `Card::bm_work` |

### Layer 4: Verus/Creusot functional

Async send → response or split-timeout or generation-expiry, exactly one terminal callback per submit. Iso queue → per-cycle callback for each completed packet, with header values matching descriptor program. Encoded as Verus refinement of an abstract 1394 transaction-state-machine spec.
