# Tier-3: drivers/hv/ — Hyper-V guest stack overview

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/hv/00-overview.md
upstream-paths:
  - drivers/hv/
  - drivers/hv/vmbus_drv.c
  - drivers/hv/connection.c
  - drivers/hv/channel.c
  - drivers/hv/channel_mgmt.c
  - drivers/hv/ring_buffer.c
  - drivers/hv/hv.c
  - drivers/hv/hv_common.c
  - include/linux/hyperv.h
  - include/hyperv/hvhdk*.h
-->

## Summary

`drivers/hv/` implements the Linux *guest* side of Microsoft's Hyper-V hypervisor protocol — the host-to-guest communication stack used by every Linux VM running on Hyper-V (Azure, Hyper-V Server, Windows Server, WSL2). It exposes the **VMBus** virtual bus on which paravirtual drivers attach (netvsc, storvsc, hv_utils, hv_balloon, hyperv-keyboard, hyperv_fb, hv_sock); per-VM **synthetic interrupt controller** (SynIC) with per-vCPU SINT messages + STIMERs; per-channel **ring-buffer** transports; **GPADL** (Guest Physical Address Descriptor List) for memory hand-off; and **hypercall** invocation per TLFS (Hypervisor Top-Level Functional Specification).

The same directory also hosts **mshv** — the Linux *root partition* implementation (Linux-as-Hyper-V-root) using `mshv_*.c` — which is out of scope for this Tier-3 (covered separately under a future `mshv.md`).

This overview maps the guest-side surface; per-component internals live in `vmbus.md` and future siblings (`hv-synic.md`, `hv-utils.md`, `hv-balloon.md`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vmbus_connection` | global guest-host connection state | `drivers::hv::Connection` |
| `struct vmbus_channel` | per-virtual-device channel: ringbuffer + monitor page + state | `drivers::hv::VmbusChannel` |
| `struct hv_device` / `struct hv_driver` | per-VMBus device + driver | `drivers::hv::{HvDevice,HvDriver}` |
| `struct hv_ring_buffer_info` | per-direction (send/recv) ringbuffer header + mapped pages | `drivers::hv::RingBuffer` |
| `vmbus_connect()` / `vmbus_disconnect()` | guest-host handshake, version negotiate, alloc connection pages | `Connection::connect` / `_disconnect` |
| `vmbus_negotiate_version(...)` | VERSION_WIN10_V6_0 ... selection | `Connection::negotiate_version` |
| `vmbus_request_offers()` | request OFFER_CHANNEL messages enumerating each device | `Connection::request_offers` |
| `vmbus_device_register(child)` / `_unregister(child)` | per-device kobj register on the VMBus | `VmbusDevice::register` / `_unregister` |
| `__vmbus_driver_register(drv, owner, modname)` / `vmbus_driver_unregister(drv)` | per-driver register | `VmbusDriver::register` / `_unregister` |
| `vmbus_open(chan, send_size, recv_size, userdata, len, onchannel_cb, context)` / `vmbus_close(chan)` | per-channel open/close (alloc ringbuf, establish GPADL, send OPEN_CHANNEL) | `VmbusChannel::open` / `_close` |
| `vmbus_alloc_ring(chan, send, recv)` / `vmbus_free_ring(chan)` | per-channel ringbuf alloc/free | `VmbusChannel::alloc_ring` / `_free_ring` |
| `vmbus_establish_gpadl(chan, kbuf, send, recv, gpadl)` / `vmbus_teardown_gpadl(chan, gpadl)` | per-channel GPADL setup/teardown | `VmbusChannel::gpadl_*` |
| `vmbus_sendpacket(...)` / `_sendpacket_pagebuffer(...)` / `_sendpacket_mpb_desc(...)` | per-channel send variants | `VmbusChannel::send_*` |
| `vmbus_recvpacket(...)` / `vmbus_recvpacket_raw(...)` | per-channel receive | `VmbusChannel::recv_*` |
| `vmbus_isr()` / `vmbus_on_event(data)` / `vmbus_on_msg_dpc(data)` | per-vCPU event + msg DPC drain | `Subsystem::{isr,on_event,on_msg_dpc}` |
| `hv_ringbuffer_init(info, pages, n, headers)` / `hv_ringbuffer_write(...)` / `_read(...)` | per-ringbuffer transport | `RingBuffer::{init,write,read}` |
| `hv_setup_vmbus_handler(vmbus_isr)` / `hv_remove_vmbus_handler()` | arch-glue: register guest SINT handler | `Subsystem::hv_setup` / `_remove` |

## Compatibility contract

REQ-1: VMBus uses Hyper-V synthetic interrupts (SINTs) — per-vCPU `SIMP` (Synthetic Interrupt Message Page) and `SIEFP` (Synthetic Interrupt Event Flags Page) MSRs mapped read-only into kernel; SINT2 by default routes VMBus messages to `vmbus_on_msg_dpc`.

REQ-2: Per-channel ringbuffer transport: pair of 1-page-or-more ring buffers (one send, one recv) with `hv_ring_buffer` header (read_index, write_index, interrupt_mask, pending_send_sz, feature_bits) + data area. Producer / consumer cooperate via `hv_signal_on_write` which writes to the host's monitor page if interrupts unmasked.

REQ-3: GPADL (Guest Physical Address Descriptor List): per-channel buffer hand-off encoding GPN array; host translates via VTL DMA to access guest pages; teardown via GPADL_TEARDOWN message.

REQ-4: Connection establishment via `vmbus_connect`: alloc 4-page interrupt page + 2 monitor pages, post `CHANNELMSG_INITIATE_CONTACT` with target SINT + monitor-page GPAs, host replies with VERSION_RESPONSE; negotiated version dictates supported features (VERSION_WIN10_V4_1+ supports MODIFYCHANNEL with-ack, etc.).

REQ-5: Per-channel `state` lifecycle: CHANNEL_OFFER -> CHANNEL_OPENING -> CHANNEL_OPENED -> CHANNEL_CLOSING -> CHANNEL_CLOSED; transitions driven by host OFFER / RESCIND messages + guest OPEN / CLOSE requests.

REQ-6: STIMER (Synthetic Timer): per-vCPU STIMER0..3 used by clocksource (`hyperv_timer`) + accelerated TSC; not directly in `drivers/hv/` but VMBus glues via `hv_setup_kexec_handler` + per-vCPU init.

REQ-7: Hot-add of channels: host posts `OFFER_CHANNEL` for new virtual devices at runtime (e.g. on NIC vNIC add in Azure); `vmbus_onoffer` queues to `work_queue`, eventually invokes `vmbus_device_register` on the bus.

REQ-8: Rescind path: host posts `RESCIND_OFFER_CHANNEL`; `vmbus_onoffer_rescind` queues to `rescind_work_queue`, guest detaches driver via `vmbus_device_unregister`, then GPADL torn down + ringbuf freed.

REQ-9: `vmbus_post_msg(buf, len, can_sleep)` posts a CHANNELMSG to host via `HvPostMessage` hypercall on per-vCPU input page; retries on `HV_STATUS_INSUFFICIENT_BUFFERS`.

REQ-10: Confidential VMs: when running under SEV-SNP / TDX-Hyper-V (`is_confidential`), the connection page + ringbuffer + GPADL pages must be in **shared (decrypted) memory**; cross-ref `drivers::hv::Connection::shared_pages`.

## Acceptance Criteria

- [ ] AC-1: Linux boots in a Hyper-V guest (Azure VM image); `dmesg | grep -i 'hv_vmbus\|Hyper-V\|VMBus'` shows connection established + version negotiated.
- [ ] AC-2: Per-device probes: netvsc, storvsc, hv_utils, hv_balloon all bind via `vmbus_driver_register` + `vmbus_device_register` flow.
- [ ] AC-3: Network + storage IO functional on Azure (synthetic netvsc + storvsc paths).
- [ ] AC-4: Hot-add: attach a new vNIC in Azure UI; netvsc driver attaches to new channel at runtime via OFFER_CHANNEL path.
- [ ] AC-5: Suspend/resume on Azure-supported guests (`vmbus_bus_suspend` / `_resume`) preserves per-channel state.
- [ ] AC-6: Confidential VM boot (SEV-SNP guest on Azure CVM): shared-pages path used; ringbuffers in decrypted memory.

## Architecture

Boot order:

1. Arch glue (`arch/x86/kernel/cpu/mshyperv.c` or `arch/arm64/hyperv/`) detects Hyper-V hypervisor via CPUID, sets `hv_root_partition = false`, calls `hv_setup_vmbus_handler(vmbus_isr)` to install per-vCPU SINT2 handler.
2. `vmbus_bus_init` (in `vmbus_drv.c`): register `vmbus_bus_type` with the driver core, create `/sys/bus/vmbus/` + `/sys/devices/<channel-guid>/`.
3. `vmbus_connect`: post INITIATE_CONTACT, wait for VERSION_RESPONSE. Multiple versions attempted in `vmbus_versions[]` order until one accepted.
4. `vmbus_request_offers`: post REQUEST_OFFERS; host floods OFFER_CHANNEL messages; each enqueued to `vmbus_connection.work_queue`.
5. Per-OFFER: `vmbus_process_offer` allocates `vmbus_channel`, computes GUID-derived device + instance ID, calls `vmbus_device_register` -> `device_register` -> `vmbus_bus_match` -> binds matching `hv_driver`.

Per-channel lifecycle:

```
guest                        host
  ----- INITIATE_CONTACT  ----->
  <---- VERSION_RESPONSE   -----
  ----- REQUEST_OFFERS    ----->
  <---- OFFER_CHANNEL     -----  (one per virtual device)
  ...                           (vmbus_process_offer -> vmbus_device_register)
  <---- ALL_OFFERS_DELIVERED ---
                                (driver binds, calls vmbus_open)
  ----- OPEN_CHANNEL      ----->
  <---- OPEN_CHANNEL_RESULT ---
                                (data plane: ringbuf send/recv)
  <---- RESCIND_OFFER_CHANNEL --
                                (vmbus_device_unregister)
  ----- CLOSE_CHANNEL     ----->
```

ISR path (`vmbus_isr`):

1. Read per-vCPU `vmbus_evt` percpu var (set by arch SINT handler).
2. For each set bit in event flags page: locate channel by sub-channel-index, schedule `vmbus_chan->callback_mode == HV_CALL_BATCHED` tasklet (`vmbus_on_event`) or directly invoke `onchannel_callback`.
3. Drain SIMP message slot (SINT2): for each VMBus message, queue to `msg_dpc` tasklet, invoke `vmbus_on_msg_dpc` -> `channel_message_table[msgtype].message_handler`.

Ringbuffer transport (`ring_buffer.c`):

- `hv_ringbuffer_write`: lock-free MPSC (multiple kernel producers on same channel must serialize via `chan->outbound_lock`), single host consumer; check available space, `hv_copyto_ringbuffer` (memcpy with circular wrap), advance write_index, `hv_signal_on_write` posts to host monitor page if interrupt_mask clear AND host's pending_send_sz exceeded.
- `hv_ringbuffer_read`: single guest consumer (per-channel callback), packets parsed by `hv_pkt_iter_first` / `_next` / `_close`.

## Hardening

VMBus is the *trust boundary between the guest kernel and the hypervisor*. The hypervisor is sovereign and can already read/write all guest pages, so the boundary is asymmetric — but the *kernel-internal* surfaces (channel message dispatch, ringbuffer parsing, OFFER processing, GPADL handling, hypercall return-value handling) are reachable by any process that can post malformed messages via hv_sock or that influences a paravirtualized device under Azure live-migration. All hypervisor-supplied data (OFFER fields, message types, channel ringbuffer headers) must be treated as untrusted.

The SIMP (Synthetic Interrupt Message Page) and SIEFP (Event Flags Page) are MSR-mapped per-vCPU; once initialized they must be re-mapped `PAGE_KERNEL_RO` for the kernel side (host writes go through hypercall semantics, not pagetable). Confidential-VM paths require these pages in shared memory.

Hypercall invocations (`hv_do_hypercall`, `hv_do_fast_hypercall`) accept opcode + input/output GPA; per-TLFS-allowlist validation of opcode is the structural defense (refuse opcodes not on the guest-permitted list).

`vmbus_post_msg` writes guest-page data the hypervisor parses; defense-in-depth requires bounding `len` to `HV_MESSAGE_PAYLOAD_BYTE_COUNT` and rejecting any caller buffer whose VA falls outside kernel direct-map.

Per-channel `hv_signal_on_write` toggles host wakeup; spurious signaling is benign but worst-case suppression is a data-plane wedge — defend via Verus per-channel invariant.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `vmbus_channel`, `hv_device`, `hv_driver`, `hv_ring_buffer_info`, GPADL descriptor slabs; user-mode reads of `/sys/bus/vmbus/devices/*` strictly bounded.
- **PAX_KERNEXEC** — `vmbus_bus_type.{match,probe,remove,uevent,shutdown}`, `hv_driver.{probe,remove,suspend,resume,shutdown}`, and `channel_message_table[].message_handler` all placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize stack across `vmbus_isr`, `vmbus_on_event`, `vmbus_on_msg_dpc`, `hv_ringbuffer_write`/`_read`, `vmbus_open`, `vmbus_device_register`.
- **PAX_REFCOUNT** — saturating refcount on `vmbus_channel.probe_done`, per-channel `hv_device.kobj.kref`, `vmbus_connection`-level worker queue references.
- **PAX_MEMORY_SANITIZE** — zero-on-free on `vmbus_channel`, GPADL descriptor lists, ringbuffer backing pages on teardown, and message-info slab so stale channel state (host secrets, prior ring contents) cannot bleed across re-offer.
- **PAX_UDEREF** — SMAP/PAN enforced on `/dev/vmbus`/`hv_sock` paths (covered in `net/vmw_vsock/hyperv_transport.c`) and any uapi entry that lands here.
- **PAX_RAP / kCFI** — `vmbus_bus_type` ops, `hv_driver` ops, `channel_message_table` per-message handlers, and per-channel `onchannel_callback` typed under kCFI; `__ro_after_init` vtables.
- **GRKERNSEC_HIDESYM** — gate `vmbus_channel` / `hv_device` kernel pointer disclosure in `/sys/bus/vmbus/`, `/proc/interrupts`, debugfs behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — suppress Hyper-V probe banners (CPUID, hypervisor leaf, partition properties, build ID disclosed via VERSION_RESPONSE) from non-CAP_SYSLOG dmesg.
- **SIMP / SIEFP page protection** — after per-vCPU init, remap SIMP + SIEFP `PAGE_KERNEL_RO` from the guest-kernel side; refuse re-init under run-time CPU hotplug without explicit teardown.
- **Hypercall TLFS allowlist** — `hv_do_hypercall` / `hv_do_fast_hypercall` consult a build-time allowlist of TLFS opcodes the guest is permitted to issue; refuse opcodes outside the allowlist (defense vs. driver-misuse hypercall escalation).
- **Confidential-VM shared-page validation** — on SEV-SNP / TDX-Hyper-V guests, the connection page, monitor pages, GPADL pages, and ringbuffer pages must reside in shared/decrypted memory; refuse to bring up VMBus otherwise.
- **OFFER field validation** — `vmbus_onoffer` validates per-OFFER fields (mmio_ranges, sub_channel_index, monitor_id, target_vp) against TLFS bounds before allocating `vmbus_channel`.
- **Message-type allowlist** — `channel_message_table[msgtype]` strictly bounded by `CHANNELMSG_COUNT`; unknown msgtype dropped with rate-limited log.

Rationale: VMBus is the seam between a Linux guest and the Hyper-V hypervisor. The hypervisor is trusted in the legacy model but on confidential VMs it explicitly is not; even on classic Hyper-V, malformed OFFER / RESCIND / hot-add messages can be injected by a co-tenant compromising the host control plane. Marking SIMP/SIEFP RO post-init, allowlisting hypercall opcodes, validating OFFER fields against TLFS, sanitizing channel state on rescind, and gating dmesg banners turns the boundary from "trust the host" into "trust the host but contain the host" — the model the confidential-VM era requires.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VMBus core internals (covered in `vmbus.md`)
- Per-device drivers (netvsc, storvsc, hv_balloon, hv_utils — covered under respective `drivers/net/`, `drivers/scsi/`, etc.)
- mshv (Linux-as-root-partition) — covered in future `mshv.md`
- hv_sock transport (covered in `net/vmw_vsock/hyperv_transport.c`)
- Hyper-V clocksource (`drivers/clocksource/hyperv_timer.c`)
- 32-bit-only paths
- Implementation code
