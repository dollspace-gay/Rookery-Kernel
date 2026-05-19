# Tier-3: drivers/hv/vmbus_drv.c + channel.c + connection.c + channel_mgmt.c — VMBus core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/hv/00-overview.md
upstream-paths:
  - drivers/hv/vmbus_drv.c
  - drivers/hv/channel.c
  - drivers/hv/channel_mgmt.c
  - drivers/hv/connection.c
  - drivers/hv/ring_buffer.c
  - drivers/hv/hyperv_vmbus.h
  - include/linux/hyperv.h
-->

## Summary

VMBus is Hyper-V's per-VM virtual bus carrying paravirt devices to the Linux guest. This Tier-3 covers the four core files:

- `vmbus_drv.c` (~3060 lines) — bus + driver model, sysfs, MMIO arbiter, suspend/resume, ISR + msg DPC, panic notifier.
- `connection.c` (~531 lines) — guest-host handshake, version negotiation, `vmbus_post_msg`, `vmbus_set_event`.
- `channel.c` (~1298 lines) — per-channel open/close, GPADL establish/teardown, send/recv variants, request-id table.
- `channel_mgmt.c` (~1687 lines) — OFFER / RESCIND processing, channel allocation, sub-channel multiplexing.
- `ring_buffer.c` (~654 lines) — per-direction lock-free ringbuffer producer/consumer (see also `hv/00-overview.md`).

This is the structural surface every paravirt device driver (netvsc / storvsc / hv_balloon / hv_utils / hv_sock) builds atop.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `vmbus_bus_type` | `struct bus_type` for the VMBus | `drivers::hv::VmbusBusType` |
| `vmbus_device_register(child)` | per-device probe attach | `VmbusDevice::register` |
| `vmbus_device_unregister(dev)` | per-device detach | `VmbusDevice::unregister` |
| `__vmbus_driver_register(drv, owner, modname)` | per-driver register | `VmbusDriver::register` |
| `vmbus_open(chan, send, recv, ud, ud_len, cb, ctx)` | per-channel open: alloc ringbuf + GPADL + OPEN_CHANNEL msg | `VmbusChannel::open` |
| `vmbus_close(chan)` | CLOSE_CHANNEL + tear ringbuf | `VmbusChannel::close` |
| `vmbus_alloc_ring(chan, send_sz, recv_sz)` / `vmbus_free_ring(chan)` | allocate per-direction ringbuf pages | `VmbusChannel::{alloc,free}_ring` |
| `vmbus_establish_gpadl(chan, kbuf, send_sz, recv_sz, &gpadl)` / `vmbus_teardown_gpadl(chan, &gpadl)` | per-buffer GPADL register/unregister | `VmbusChannel::gpadl_*` |
| `vmbus_send_modifychannel(chan, target_vp)` | retarget channel events to a different vCPU | `VmbusChannel::modify_channel` |
| `vmbus_sendpacket(chan, buf, len, rqid, type, flags)` / `_pagebuffer` / `_mpb_desc` | per-channel send variants | `VmbusChannel::send_{packet,pagebuffer,mpb}` |
| `vmbus_recvpacket(chan, buf, len, &len_out, &rqid)` | per-channel recv | `VmbusChannel::recv_packet` |
| `vmbus_alloc_requestor(rqstor, size)` / `_free_requestor(rqstor)` | per-channel request-id table for async match | `VmbusChannel::requestor_*` |
| `vmbus_setevent(chan)` | post per-channel monitor-page event | `VmbusChannel::set_event` |
| `vmbus_set_event(chan)` | post HvSignalEvent fast hypercall | `Connection::set_event` |
| `vmbus_post_msg(buf, len, can_sleep)` | HvPostMessage hypercall wrapper | `Connection::post_msg` |
| `vmbus_negotiate_version(msginfo, ver)` | version-handshake state machine | `Connection::negotiate_version` |
| `vmbus_isr()` | per-vCPU ISR (per-vCPU SINT2 + event flags) | `Subsystem::isr` |
| `vmbus_on_event(data)` | per-channel event tasklet | `Subsystem::on_event` |
| `vmbus_on_msg_dpc(data)` | per-msg DPC tasklet | `Subsystem::on_msg_dpc` |
| `vmbus_onoffer(hdr)` / `vmbus_onoffer_rescind(hdr)` | OFFER / RESCIND handlers | `Subsystem::{onoffer,rescind}` |
| `hv_ringbuffer_init(info, pages, n, hdr_pages)` | per-direction ringbuf init | `RingBuffer::init` |
| `hv_ringbuffer_write(chan, kvecs, n, rqid, xpg)` / `_read(chan, buf, len, &len_out, &rqid, raw)` | ringbuf data plane | `RingBuffer::{write,read}` |
| `hv_pkt_iter_first(chan)` / `__hv_pkt_iter_next(chan, p)` / `hv_pkt_iter_close(chan)` | per-recv packet iteration | `RingBuffer::pkt_iter_*` |

## Compatibility contract

REQ-1: VMBus device IDs are 16-byte GUIDs (`hv_device.dev_type`); matching via `vmbus_bus_match` compares against per-driver `hv_driver.id_table[].guid`.

REQ-2: Per-channel ringbuffer: pair of N-page pages (N tunable per device; netvsc 16, storvsc 32, hv_balloon 1); `hv_ring_buffer` header at offset 0; data area follows. Header carries `read_index`, `write_index`, `interrupt_mask`, `pending_send_sz`, `feature_bits` (FEAT_PENDING_SEND_SZ).

REQ-3: GPADL establish: build `vmbus_channel_gpadl_header` message carrying GPN list (4 KB units) of buffer pages; host returns GPADL handle; guest stores in `vmbus_gpadl.gpadl_handle`. Teardown: send `CHANNELMSG_GPADL_TEARDOWN` with handle, wait completion.

REQ-4: Per-channel `state` machine: CHANNEL_OFFER_STATE (post-OFFER, pre-open) -> CHANNEL_OPENING_STATE (post-OPEN_CHANNEL msg, pre-OPEN_CHANNEL_RESULT) -> CHANNEL_OPENED_STATE -> CHANNEL_CLOSING_STATE -> CHANNEL_CLOSED_STATE.

REQ-5: Per-channel monitor-page bitmap: each channel has `monitor_grp` (0-3) + `monitor_bit` (0-31); host monitors monitor pages 0/1 for guest-to-host signals; guest monitors event-flag page for host-to-guest signals.

REQ-6: Per-channel sub-channels: device driver can request additional channels via `vmbus_request_offers` after primary open; sub-channels share device-type GUID but distinct sub-channel-index; used by netvsc multi-queue + storvsc multi-LUN.

REQ-7: Per-channel request-id table (`vmbus_requestor`): bitmap-tracked `u64` request ids; producer claims id, embeds in packet trans_id, consumer matches response to producer wait. Defends async out-of-order responses.

REQ-8: Per-channel `target_vp`: guest can retarget channel interrupts via `CHANNELMSG_MODIFYCHANNEL` (and ack variant on VERSION_WIN10_V4_1+); used for IRQ balancing.

REQ-9: Suspend/resume (`vmbus_bus_suspend` / `_resume`): per-channel state saved; on resume re-establish via re-OPEN sequence; in-flight requestor entries flushed.

REQ-10: Confidential VMs (`is_confidential`): connection page, monitor pages, GPADL pages, ringbuffer pages all in shared/decrypted memory; `vmbus_set_event` may use `HvSignalEvent` hypercall directly rather than monitor page.

## Acceptance Criteria

- [ ] AC-1: VMBus connection establishes on Azure / Hyper-V guest: `dmesg | grep 'Vmbus version'` shows negotiated version (WIN10_V6_0 on recent hosts).
- [ ] AC-2: `/sys/bus/vmbus/devices/` enumerates one entry per offered device (netvsc, storvsc, hv_utils, etc.).
- [ ] AC-3: Per-channel sysfs (id, state, monitor_id, class_id, device_id, in/out ring_buffer_pages, interrupts, events, monitor_pending) populated and readable.
- [ ] AC-4: `vmbus_open` from netvsc allocates ring + GPADL + sends OPEN_CHANNEL + returns; subsequent `vmbus_sendpacket` + ringbuf signal triggers host-side processing visible as RX traffic.
- [ ] AC-5: Hot-add: attach a new vNIC; OFFER_CHANNEL arrives; `vmbus_process_offer` registers `hv_device`; netvsc binds.
- [ ] AC-6: Rescind: host sends RESCIND_OFFER_CHANNEL; `vmbus_onoffer_rescind` queues to rescind_work_queue; driver detached; GPADL torn down; channel freed; no leak per kmemleak.
- [ ] AC-7: Suspend/resume cycle on Azure: VM suspends to host, resumes; ringbuffers re-established; in-flight requestor entries flushed; no requestor-id collision.

## Architecture

`vmbus_bus_init`:

1. Register `vmbus_bus_type` with driver core.
2. Allocate `vmbus_connection.work_queue` (single-threaded WQ for OFFER processing), `_handle_primary_chan_wq`, `_handle_sub_chan_wq`, `rescind_work_queue` (separate WQ to avoid OFFER<->RESCIND ordering deadlocks).
3. `hv_synic_alloc` per-vCPU: alloc SIMP + SIEFP + event-flags + message-page + post-msg-page.
4. CPU hotplug callback `hv_synic_init` per-online-CPU writes the SIMP/SIEFP MSRs.
5. `hv_setup_vmbus_handler(vmbus_isr)` registers per-vCPU SINT handler.
6. `vmbus_connect` posts INITIATE_CONTACT -> wait VERSION_RESPONSE.
7. `vmbus_request_offers` floods host -> per-OFFER `vmbus_onoffer` queues work to `work_queue` -> `vmbus_process_offer` allocates channel + `hv_device` + `device_register` -> driver-core matches against registered `hv_driver`s.

Per-channel `vmbus_open(chan, send_sz, recv_sz, ud, ud_len, cb, ctx)`:

1. Validate `chan->state == CHANNEL_OPEN_STATE`; transition to CHANNEL_OPENING.
2. `vmbus_alloc_ring(chan, send_sz, recv_sz)` -> `alloc_pages(GFP_KERNEL, order)` for combined send+recv pages.
3. `hv_ringbuffer_init` initializes per-direction headers (read_index=write_index=0, interrupt_mask=0, feature_bits=PENDING_SEND_SZ).
4. `vmbus_establish_gpadl(chan, ringbuf_kva, send_sz, recv_sz, &chan->ringbuffer_gpadlhandle)`:
   - Build `CHANNELMSG_GPADL_HEADER` with GPN list; if list spans, additional `CHANNELMSG_GPADL_BODY` messages.
   - Wait for `CHANNELMSG_GPADL_CREATED` completion.
5. Build `CHANNELMSG_OPENCHANNEL` carrying ringbuffer GPADL handle + send_size_in_pages + downstream_ringbuffer_pageoffset + target_vp + user_data + connection_id.
6. `vmbus_post_msg` -> wait `CHANNELMSG_OPENCHANNEL_RESULT`. On non-zero status: tear GPADL, free ring, return error.
7. Set `chan->onchannel_callback = cb`, `chan->channel_callback_context = ctx`; `chan->state = CHANNEL_OPENED_STATE`.
8. `vmbus_alloc_requestor(&chan->requestor, ...)` for the request-id table (size from `chan->max_pkt`).

ISR + msg DPC:

```
arch SINT2 handler (per-vCPU)
   -> set vmbus_evt percpu flag
   -> vmbus_isr (via hv_setup_vmbus_handler)
      -> __vmbus_isr
         -> if SIMP slot 0 holds message: schedule msg DPC tasklet (per-cpu)
            -> vmbus_on_msg_dpc(data)
               -> channel_message_table[msgtype].message_handler(hdr)
                  -> e.g. vmbus_onoffer, vmbus_onoffer_rescind, vmbus_onopen_result, ...
         -> for each set bit in event_flags page: per-channel callback
            -> if HV_CALL_BATCHED: tasklet_schedule(&chan->callback_event)
               -> vmbus_on_event -> chan->onchannel_callback(chan->channel_callback_context)
            -> if HV_CALL_ISR: chan->onchannel_callback(...) directly
```

Sub-channel multiplex (`vmbus_alloc_channel` invoked for primary + each sub-channel; primary's `subchannel_list` linked list anchors sub-channels; `chan->primary_channel` back-pointer on sub-channels).

Rescind (`vmbus_onoffer_rescind`):

1. Lookup channel by relid; mark `chan->rescind = true`.
2. Queue to `rescind_work_queue` -> `vmbus_rescind_channel_work`:
   - `device_unregister(&hv_dev->device)` -> driver `remove` called -> driver calls `vmbus_close`.
   - `free_channel(chan)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `channel_state_no_skip` | INVARIANT | per-channel state transitions only along legal edges (OFFER -> OPENING -> OPENED -> CLOSING -> CLOSED) |
| `gpadl_no_double_teardown` | UAF | `vmbus_teardown_gpadl` called at most once per `vmbus_gpadl` handle |
| `requestor_no_id_collision` | UNIQUENESS | per-channel request-id bitmap never issues an outstanding id twice |
| `ringbuf_no_oob_write` | OOB | `hv_ringbuffer_write` respects ringbuf size; circular wrap correct |
| `ringbuf_no_oob_read` | OOB | `hv_ringbuffer_read` bounds packet header parse + payload copy by buffer size |

### Layer 2: TLA+

`models/hv/vmbus_handshake.tla` (parent-declared): proves INITIATE_CONTACT / VERSION_RESPONSE / REQUEST_OFFERS / ALL_OFFERS_DELIVERED handshake terminates under host-version-mismatch retry; `models/hv/channel_lifecycle.tla` proves OFFER/OPEN/CLOSE/RESCIND ordering preserves CHANNEL_OPENED <-> CHANNEL_CLOSED invariant under concurrent rescind during open.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| Post `vmbus_open`: `chan->state == CHANNEL_OPENED_STATE` AND GPADL handle valid AND ringbuffer mapped | `VmbusChannel::open` |
| Post `vmbus_close`: GPADL torn down AND ringbuffer freed AND `chan->state == CHANNEL_CLOSED_STATE` AND requestor empty | `VmbusChannel::close` |
| Post `vmbus_sendpacket`: write-index advanced by exactly `len + sizeof(vmpacket_descriptor) + alignment`; signal posted IFF interrupt unmasked AND pending_send_sz threshold crossed | `VmbusChannel::send_packet` |
| Post `vmbus_alloc_requestor`: bitmap zeroed AND id 0 reserved AND `pin_lock` initialized | `VmbusChannel::alloc_requestor` |

### Layer 4: Verus functional

End-to-end: OFFER arrival -> `vmbus_process_offer` -> `vmbus_device_register` -> driver `probe` -> `vmbus_open` -> driver sends packets via `vmbus_sendpacket` -> host responds via ringbuffer -> driver `onchannel_callback` reads via `vmbus_recvpacket_raw` + `hv_pkt_iter_*` -> matches request-id -> wakes producer.

## Hardening

VMBus message dispatch (`channel_message_table[msgtype].message_handler`) is indirect-typed and untrusted-data-driven — host can send any msgtype it likes. Today `vmbus_on_msg_dpc` checks `msgtype < CHANNELMSG_COUNT`; kCFI adds typesig matching. Per-message-handler input is `struct vmbus_channel_message_header*` whose payload is variable-typed by msgtype; each handler MUST cast to the right specific struct and bounds-check by `msginfo->msg_len`.

GPADL teardown is delicate: the host MUST acknowledge GPADL_TEARDOWN before the guest can free the underlying pages; if the guest frees too early, host DMA continues against freed pages. `vmbus_teardown_gpadl` correctly waits the ack, but the failure path (host wedged) must time out and either retain the pages forever OR (only on shutdown) recycle them with a loud warning. Capture as Verus precondition.

`vmbus_alloc_requestor` allocates a bitmap-backed `u64[]` table; size is per-channel-driver-supplied. Bound the size to `MAX_REQUESTOR_SIZE` (typically `chan->max_pkt`) to defeat per-channel-driver-mistake denial.

Monitor pages are MMIO-backed (host monitors guest writes) — they must remain present (no page-out) for the channel's lifetime. PFN-based mapping via `hv_setup_pre_dispatch` (arch glue) handles the pin.

Per-channel `outbound_lock` serializes multiple kernel producers on the same channel. `hv_ringbuffer_write` is invoked under this lock; any path that calls it without the lock is a data corruption bug. Encode as Verus precondition.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `vmbus_channel`, `hv_device`, `vmbus_requestor`, `vmbus_gpadl`, ringbuffer-info, and OPEN/OFFER message slabs; copy_to_user of `/sys/bus/vmbus/devices/*` bounded.
- **PAX_KERNEXEC** — `vmbus_bus_type` ops, `hv_driver` ops, `channel_message_table[].message_handler`, per-channel `onchannel_callback` table all in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize stack across `vmbus_isr`, `vmbus_on_msg_dpc`, `vmbus_on_event`, `vmbus_open`, `vmbus_close`, `vmbus_sendpacket`, `vmbus_recvpacket`.
- **PAX_REFCOUNT** — saturating refcount on `hv_device.kobj.kref`, `vmbus_channel` (channel_id-keyed lookup must not UAF on rescind race), and per-`vmbus_connection` work-queue references.
- **PAX_MEMORY_SANITIZE** — zero-on-free on per-channel ringbuffer pages on `vmbus_free_ring`, on GPADL descriptor pages on teardown, on `vmbus_requestor.req_arr` on free, and on `vmbus_channel_msginfo` slab; sanitize defeats stale guest data on re-offer.
- **PAX_UDEREF** — SMAP/PAN on per-channel sysfs writers + on hv_sock entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `channel_message_table[].message_handler` (per-msgtype dispatch), `hv_driver.{probe,remove,suspend,resume}`, `onchannel_callback`, and `vmbus_bus_type.{match,probe,remove}` typed under kCFI.
- **GRKERNSEC_HIDESYM** — gate per-channel `class_id`, `device_id`, `monitor_id`, ringbuffer GPA, and channel kernel pointers in `/sys/bus/vmbus/` behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict VMBus probe banners (version, channel-relid disclosure, host build ID) to CAP_SYSLOG; rate-limit OFFER / RESCIND logs.
- **Channel ringbuffer PAX_USERCOPY** — debugfs ringbuffer dumps (`/sys/kernel/debug/hyperv/<dev>/`) bounded by ringbuf size; refuse oversize reads.
- **Monitor pages RO post-init** — once SIMP / SIEFP / monitor pages mapped, mark guest pagetable entries RO from kernel side (host writes via hypercall semantics, not pagetable).
- **Channel hot-add CAP_SYS_ADMIN** — sysfs hot-add / driver-override / unbind for VMBus devices requires CAP_SYS_ADMIN in init user namespace.
- **Channel-id PAX_REFCOUNT** — `relid_to_channel[relid]` lookup table guarded by saturating refcount + RCU; rescind race cannot UAF a freed channel while in-flight ISR holds the relid.
- **GPADL teardown sanitize** — backing pages overwritten on `vmbus_teardown_gpadl` ack before being returned to the page allocator; defense against host-resident DMA-after-teardown.
- **OFFER field allowlist** — `vmbus_onoffer` validates `offer.if_type`, `offer.if_instance`, `monitor_id`, `target_vp`, `is_dedicated_interrupt`, `connection_id` against TLFS-permitted ranges; refuse malformed offers.
- **Sub-channel index bound** — sub-channel count per primary bounded by `chan->num_sc`; refuse OFFER carrying sub-channel-index > primary's declared limit.

Rationale: VMBus is the kernel's primary attack surface against a compromised Hyper-V control plane. Every OFFER message, every GPADL handle, every ringbuffer header field, and every channel-message-table dispatch is host-influenced. Concentrating discipline at message dispatch (kCFI + msgtype allowlist), at channel lifecycle (refcount + RCU), at GPADL teardown (sanitize + completion-wait), and at OFFER fields (TLFS bounds) transforms host-trust into structural validation — the model required when the same code path serves both classic Hyper-V and confidential-VM SEV-SNP / TDX-Hyper-V guests.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-device drivers (netvsc, storvsc, hv_utils, hv_balloon, hyperv-keyboard, hyperv_fb) — covered under respective `drivers/net/`, `drivers/scsi/`, etc. Tier-3s
- Synthetic Interrupt Controller (SynIC) glue (`hv.c`, `hv_common.c`) — covered in future `hv-synic.md`
- mshv (Linux-as-root) — covered separately under future `mshv.md`
- hv_sock transport (covered in `net/vmw_vsock/hyperv_transport.c`)
- Confidential-VM specifics (covered in future `hv-cvm.md`)
- 32-bit-only paths
- Implementation code
