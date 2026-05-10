---
title: "Tier-3: drivers/usb/core/{urb,message}.c — USB Request Block (URB) submission + completion + cancellation + sync messages"
tags: ["tier-3", "drivers-usb", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The URB (USB Request Block) is THE USB I/O abstraction — every USB I/O — keyboard report, mouse event, USB-storage SCSI command, USB-Ethernet packet, USB-audio sample, USB-camera frame, USB-CDC modem byte — is encoded as one or more URBs submitted through this code. Each URB describes: target pipe (device + ep + direction), transfer-buffer (in-kernel address + DMA-mapped address), length, type-specific bits (control setup-packet, isochronous frame-list, bulk/interrupt nothing-extra), completion callback. URB submitted async; HCD (xHCI/eHCI/oHCI/uHCI) processes; completion callback invoked from per-HCD IRQ context.

Plus the synchronous-message API (`usb_control_msg`, `usb_bulk_msg`, `usb_interrupt_msg`) — wraps `usb_submit_urb` + `wait_for_completion` for callers that want blocking semantics (most enumeration code paths, simple device queries).

This Tier-3 covers `drivers/usb/core/urb.c` (~1000 lines: URB lifecycle + submit + cancel) + `message.c` (~2500 lines: synchronous wrappers + control-message helpers + GET_DESCRIPTOR / SET_CONFIGURATION / SET_INTERFACE / CLEAR_FEATURE / SET_FEATURE / etc.).

### Acceptance Criteria

- [ ] AC-1: USB-storage data transfer (sg → URB → submit → completion → blk-mq end_request) round-trip works at expected throughput.
- [ ] AC-2: URB-anchor stress: driver submits 1000 URBs to anchor + driver-disconnect → all URBs cancelled cleanly + anchor drained < 1s.
- [ ] AC-3: Synchronous control msg test: `usb_control_msg(dev, pipe, GET_DESCRIPTOR, ...)` returns descriptor + length matches.
- [ ] AC-4: Cancel-during-IO stress: 1000x submit URB + immediately `usb_unlink_urb` → no UAF, all callbacks invoked.
- [ ] AC-5: Clear-halt test: stalled bulk endpoint cleared via `usb_clear_halt`; subsequent transfers succeed.
- [ ] AC-6: Set-configuration + set-interface flow: composite device with alt settings switches between alts correctly; per-class drivers re-bind.
- [ ] AC-7: Isochronous URB: USB-audio capture submits per-frame iso URBs; per-frame actual_length + status reported.

### Architecture

`Urb` lives in `kernel::usb::Urb`:

```
struct Urb {
  refcount: Refcount,
  dev: Arc<Device>,
  ep: Arc<HostEndpoint>,
  pipe: u32,                          // encoded type + direction + dev_addr + ep_num
  stream_id: u32,                     // for USB-3 streams
  status: AtomicI32,                  // -EINPROGRESS / 0 / -errno
  transfer_flags: u32,                 // URB_SHORT_NOT_OK / URB_ISO_ASAP / URB_NO_TRANSFER_DMA_MAP / URB_ZERO_PACKET / etc.
  transfer_buffer: NonNull<u8>,
  transfer_dma: dma_addr_t,
  sg: Option<NonNull<ScatterList>>,
  num_sgs: i32,
  num_mapped_sgs: i32,
  transfer_buffer_length: u32,
  actual_length: AtomicU32,
  setup_packet: Option<NonNull<UsbCtrlrequest>>,
  setup_dma: dma_addr_t,
  start_frame: i32,
  number_of_packets: i32,
  interval: u32,
  error_count: AtomicI32,
  context: NonNull<()>,                // per-driver opaque
  complete: fn(&Arc<Urb>),             // completion callback
  iso_frame_desc: VarLenArray<UsbIsoPacketDescriptor>,
  hcpriv: AtomicPtr<()>,                // per-HCD private state
  use_count: AtomicI32,                 // submit-in-flight count
  reject: AtomicBool,                   // rejecting new submits
  unlinked: AtomicI8,                   // unlink in progress
  anchor: Mutex<Option<Arc<UrbAnchor>>>,
  anchor_list: ListEntry,
}

struct UrbAnchor {
  urb_list: Mutex<LinkedList<Arc<Urb>>>,
  wait: WaitQueue,
  poisoned: AtomicBool,
}
```

Submit path `Urb::submit(urb, mem_flags)`:
1. Validate URB fields:
   - `urb.dev` not disconnected; `urb.ep` valid for `urb.pipe`.
   - `urb.transfer_buffer_length` non-zero ↔ `transfer_buffer` non-null OR `sg`+`num_sgs` non-zero.
   - For control: `setup_packet` non-null.
   - For iso: `number_of_packets > 0` AND `iso_frame_desc[]` populated.
2. If `urb.reject` set: return -EPERM (poisoned).
3. `urb.status.store(-EINPROGRESS)`, `urb.actual_length.store(0)`.
4. `urb.use_count.fetch_add(1)`.
5. `usb_get_urb(&urb)` (extra ref for HCD).
6. DMA-map per `transfer_flags` if not URB_NO_TRANSFER_DMA_MAP.
7. `dev.bus.hc_driver.urb_enqueue(hcd, &urb, mem_flags)` → per-HCD enqueue (xHCI: build TRBs + ring doorbell).
8. On hcd error: rollback DMA, `usb_put_urb`, `urb.use_count.fetch_sub(1)`, return errno.
9. On success: return 0; URB now in HW queue.

Completion path (called from HCD IRQ handler):
1. `usb_hcd_giveback_urb(hcd, urb, status)`:
   - DMA-unmap transfer_buffer / setup_packet.
   - `urb.status.store(status)` / `urb.actual_length.store(actual)`.
   - `urb.use_count.fetch_sub(1)`.
   - If `urb.anchor`: `urb.anchor.urb_list.remove(&urb)`; `urb.anchor.wait.wake_up_all()` if list empty.
   - `urb.complete(&urb)` (per-driver callback) — runs in IRQ/softirq context.
   - `usb_put_urb(&urb)` (HCD's ref drop).

Cancel path `Urb::unlink`:
1. `urb.unlinked.store(1)`.
2. `dev.bus.hc_driver.urb_dequeue(hcd, &urb, -ECONNRESET)` → per-HCD removes from HW queue.
3. Return immediately (non-blocking).
4. Eventually HCD's giveback path invokes `urb.complete(&urb)` with status = -ECONNRESET.

`Urb::kill`:
1. `urb.unlinked.store(1)`.
2. `dev.bus.hc_driver.urb_dequeue(hcd, &urb, -ENOENT)`.
3. `wait_event(urb.wait, urb.use_count.load() == 0)` — block until callback invoked.

`Urb::poison`:
1. `urb.reject.store(true)`.
2. `urb.kill()`.
3. Subsequent `Urb::submit` returns -EPERM.

Sync message wrapper `Device::control_msg(dev, pipe, request, requesttype, value, index, data, size, timeout)`:
1. Allocate URB + setup_packet.
2. Initialize completion struct.
3. `urb.complete = sync_msg_complete` (signals completion struct).
4. `urb.context = &completion`.
5. `Urb::submit(&urb, GFP_KERNEL)`.
6. `wait_for_completion_timeout(&completion, timeout)`.
7. If timeout: `Urb::kill(&urb)`; return -ETIMEDOUT.
8. Return urb.actual_length OR urb.status.

`Device::set_configuration(dev, configuration)`:
1. Validate `configuration` is in dev's config list.
2. SET_CONFIGURATION control msg.
3. Tear down old config's interface registrations.
4. For each interface in new config:
   - Allocate `usb_interface` struct.
   - `device_initialize(&intf.dev)` + `intf.dev.bus = &usb_bus_type` + `intf.dev.parent = &dev.dev`.
   - `device_add(&intf.dev)` triggers per-class probe.
5. Update `dev.actconfig`.

URB-anchor batch-cancel `UrbAnchor::kill_all(anchor)`:
1. While `!anchor.urb_list.is_empty()`:
   - Pop head urb.
   - `Urb::kill(&urb)`.
2. `anchor.wait.wake_up_all()`.

### Out of Scope

- Per-HCD URB enqueue/dequeue (covered in `drivers/usb/host-xhci.md` parent + `host-{ehci,ohci,uhci}.md` future Tier-3s)
- Per-driver lifecycle (covered in `drivers/usb/core-driver.md` future Tier-3)
- usbfs `/dev/bus/usb/` chardev (covered in `drivers/usb/core-devio.md` future Tier-3)
- Per-class drivers (storage / serial / hid / etc.)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct urb` | per-I/O request control block | `kernel::usb::Urb` |
| `usb_alloc_urb(iso_packets, mem_flags)` | allocate URB | `Urb::alloc` |
| `usb_free_urb(urb)` | drop refcount → release on 0 | `Urb::put` (Drop) |
| `usb_get_urb(urb)` | refcount get | `Arc::clone` |
| `usb_init_urb(urb)` | embedded-URB init | `Urb::init_in_place` |
| `usb_submit_urb(urb, mem_flags)` | submit URB to HCD | `Urb::submit` |
| `usb_unlink_urb(urb)` | non-blocking cancel | `Urb::unlink` |
| `usb_kill_urb(urb)` | blocking cancel + wait | `Urb::kill` |
| `usb_poison_urb(urb)` | mark URB poisoned (perma-cancel + reject re-submit) | `Urb::poison` |
| `usb_unpoison_urb(urb)` | inverse | `Urb::unpoison` |
| `usb_block_urb(urb)` | block all submits via this URB | `Urb::block` |
| `usb_unblock_urb(urb)` | unblock | `Urb::unblock` |
| `usb_kill_anchored_urbs(anchor)` / `_poison_*` / `_scuttle_*` | per-anchor batch cancel | `UrbAnchor::kill_all` / `_poison_all` |
| `usb_anchor_urb(urb, anchor)` / `_unanchor_urb(urb)` | anchor mgmt | `UrbAnchor::add` / `_remove` |
| `usb_wait_anchor_empty_timeout(anchor, ms)` | wait until anchor drained | `UrbAnchor::wait_empty_timeout` |
| `usb_pipe_*` family | per-pipe encode/decode helpers | `Pipe::*` |
| `usb_fill_control_urb(urb, dev, pipe, setup_packet, transfer_buffer, length, complete_fn, context)` | populate control URB | `Urb::fill_control` |
| `usb_fill_bulk_urb(urb, dev, pipe, transfer_buffer, length, complete_fn, context)` | populate bulk URB | `Urb::fill_bulk` |
| `usb_fill_int_urb(urb, dev, pipe, transfer_buffer, length, complete_fn, context, interval)` | populate interrupt URB | `Urb::fill_interrupt` |
| `usb_control_msg(dev, pipe, request, requesttype, value, index, data, size, timeout)` | sync control msg wrapper | `Device::control_msg` |
| `usb_bulk_msg(dev, pipe, data, len, &actual_length, timeout)` | sync bulk msg wrapper | `Device::bulk_msg` |
| `usb_interrupt_msg(dev, pipe, data, len, &actual_length, timeout)` | sync interrupt msg wrapper | `Device::interrupt_msg` |
| `usb_get_descriptor(dev, type, index, buf, size)` | sync GET_DESCRIPTOR | `Device::get_descriptor` |
| `usb_get_string(dev, langid, index, buf, size)` | sync GET_STRING | `Device::get_string` |
| `usb_set_configuration(dev, configuration)` | SET_CONFIGURATION + interface registration (cross-ref core-enumerate.md) | `Device::set_configuration` |
| `usb_set_interface(dev, intf, alt)` | SET_INTERFACE + alt-setting | `Device::set_interface` |
| `usb_clear_halt(dev, pipe)` | CLEAR_FEATURE(ENDPOINT_HALT) | `Device::clear_halt` |
| `usb_reset_endpoint(dev, ep)` | endpoint-only reset | `Device::reset_endpoint` |
| `usb_string(dev, index, buf, size)` | GET_STRING + sanitize | `Device::string` |

### compatibility contract

REQ-1: `struct urb` layout source-compat for in-tree drivers — every documented field (`dev`, `ep`, `pipe`, `transfer_flags`, `transfer_buffer`, `transfer_dma`, `transfer_buffer_length`, `actual_length`, `setup_packet`, `setup_dma`, `start_frame`, `number_of_packets`, `interval`, `error_count`, `context`, `complete`, `iso_frame_desc[]`) preserved.

REQ-2: URB transfer types: control (setup + data + status stages), bulk (one or many MaxPacket-sized chunks), interrupt (periodic poll at endpoint's bInterval), isochronous (per-frame data with per-frame status).

REQ-3: `usb_submit_urb` semantics:
- Validate URB fields (transfer_buffer non-null if length > 0, transfer_flags consistent, etc.).
- Per-pipe direction matches URB type.
- Return 0 on accepted (URB queued); -errno on validation failure.
- Completion callback called exactly once after submit, even on error (with urb->status reflecting reason).

REQ-4: `usb_unlink_urb` — non-blocking cancel; callback still called with `status = -ECONNRESET`.

REQ-5: `usb_kill_urb` — blocking cancel; waits for callback to have been called; callback called with `status = -ENOENT` (or completion code if the URB completed before kill arrived).

REQ-6: URB-anchor: per-driver `struct usb_anchor` tracks a set of submitted URBs; on driver-disconnect, `usb_kill_anchored_urbs` drains all in one batch.

REQ-7: Per-pipe encode/decode helpers byte-identical: `usb_sndctrlpipe`, `usb_rcvctrlpipe`, `usb_sndbulkpipe`, `usb_rcvbulkpipe`, `usb_sndintpipe`, `usb_rcvintpipe`, `usb_sndisocpipe`, `usb_rcvisocpipe` produce same encoded u32 pipe value.

REQ-8: Synchronous wrappers: `usb_control_msg` etc. return actual_length OR -errno (timeout / device-disconnect / cancel).

REQ-9: Synchronous message timeout default: 5000 ms for control, 5000 ms for bulk; per-call override.

REQ-10: `usb_set_configuration` performs SET_CONFIGURATION control + per-interface registration (cross-ref `core-enumerate.md`); per-interface `usb_interface` registered + per-class driver bound via `usb_bus_type::probe`.

REQ-11: `usb_set_interface(intf, alt)` issues SET_INTERFACE control + reconfigures per-interface endpoints to selected altsetting; HCD-side endpoint contexts updated.

REQ-12: `usb_clear_halt(pipe)` sends CLEAR_FEATURE(ENDPOINT_HALT) + resets HCD-side toggle for the endpoint.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `urb_no_uaf` | UAF | `Arc<Urb>` outlives all in-flight HCD reference + per-anchor reference; release waits for refcount==0. |
| `complete_called_exactly_once` | INVARIANT | per-URB completion callback called exactly once per `usb_submit_urb` (whether success, error, cancel, or poisoning). |
| `transfer_buffer_no_oob` | OOB | `urb.transfer_buffer + urb.transfer_buffer_length` validated; per-iso `iso_frame_desc[i].offset + length <= urb.transfer_buffer_length`. |
| `pipe_decode_no_oob` | OOB | per-pipe ep_num decoded < 16; dev_addr decoded < 128. |

### Layer 2: TLA+

`models/usb/urb_lifetime.tla` (parent-declared): proves urb refcount + submit + complete + cancel sequence — concurrent usb_unlink_urb + urb completion never produces double-complete or use-after-free.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Urb::submit` post: returned 0 means URB has been queued AND completion will be called exactly once; non-zero means completion will NOT be called | `Urb::submit` |
| `Urb::kill` post: completion has been called (urb.status set); urb.use_count == 0 | `Urb::kill` |
| Per-urb anchor invariant: `urb.anchor.is_some()` iff urb is in `urb.anchor.urb_list` | `UrbAnchor::add` / `_remove` |

### Layer 4: Verus/Creusot functional

`Urb::submit(urb) → HCD enqueue → HW processes → HCD giveback → urb.complete(urb)` round-trip equivalence: completion callback's urb has `status` and `actual_length` correctly reflecting HW completion.

### hardening

(Inherits row-1 features from `drivers/usb/00-overview.md` § Hardening.)

urb-specific reinforcement:

- **Per-URB refcount saturating** — overflow saturates at u32::MAX; defense against ref-count-overflow attack.
- **Per-URB validation at submit** — every field checked (transfer_buffer non-null, length non-zero, pipe consistent with type, etc.); defense against malformed URB from buggy driver causing kernel panic.
- **Synchronous-msg timeout-cancel + wait** ensures no leaked URB on timeout.
- **Per-anchor batch-kill** ensures all in-flight URBs drained on driver-disconnect; defense against driver-removal leaving URBs queued in HCD.
- **DMA-map / DMA-unmap paired** via per-URB transfer_flags tracking; defense against double-unmap or leaked DMA mapping.
- **Per-URB poison flag** prevents re-submission during driver-disconnect window; defense against TOCTOU between disconnect-detection and final URB-cancel.
- **Per-URB completion-callback context-aware** — callback may run from IRQ or softirq context; per-driver responsibility documented.

