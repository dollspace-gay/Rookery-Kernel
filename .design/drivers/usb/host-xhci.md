# Tier-3: drivers/usb/host/xhci-*.c — xHCI host controller (USB 2/3/4 — TRBs + slot/ep contexts + ring management)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/usb/00-overview.md
upstream-paths:
  - drivers/usb/host/xhci.c
  - drivers/usb/host/xhci.h
  - drivers/usb/host/xhci-mem.c
  - drivers/usb/host/xhci-ring.c
  - drivers/usb/host/xhci-hub.c
  - drivers/usb/host/xhci-pci.c
  - drivers/usb/host/xhci-plat.c
-->

## Summary

xHCI = eXtensible Host Controller Interface — the host-side controller spec for USB 2.0 / 3.0 / 3.1 / 3.2 / 4 (USB 4 too, gen-3+ via xHCI 1.2). Single host controller drives both LS/FS/HS (USB-2-protocol) and SS/SS+ (USB-3-protocol) ports, eliminating the need for separate eHCI + UHCI/oHCI controllers.

Architecture (per xHCI 1.2 spec):
- **Host Memory Pages** = device-context array (per-slot 64B/96B context: slot context + 31 endpoint contexts), ring buffers (command, event, transfer per endpoint).
- **Slots** = per-USB-device control block; allocated via Enable-Slot command.
- **Endpoint Contexts** = per-endpoint TRB ring + state.
- **TRBs (Transfer Request Blocks)** = 16-byte command/transfer/event entries.
- **Doorbells** = per-slot+per-ep MMIO write triggers HW to fetch new TRBs.
- **Event Ring** = HW-posted completion events (HW-owned producer, kernel-owned consumer; cycle-bit for wraparound).

This Tier-3 covers `xhci.c` (~5700 lines: per-host probe + sw init + RTPM + DCI), `xhci.h` (header), `xhci-mem.c` (device-context / TRB-ring memory mgmt), `xhci-ring.c` (~4500 lines: TRB enqueue / dequeue / event processing — the biggest hot path), `xhci-hub.c` (root-hub emulation as USB hub for core/hub.c), `xhci-pci.c` (PCIe binding), `xhci-plat.c` (platform binding for SoC xHCI).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct xhci_hcd` | per-controller control block | `kernel::usb::host::xhci::Xhci` |
| `struct xhci_op_regs` / `_run_regs` / `_cap_regs` / `_doorbell_array` / `_intr_reg_set` | per-controller MMIO register layout | `xhci::regs::*` |
| `struct xhci_slot_ctx` / `_ep_ctx` / `_input_control_ctx` / `_input_ctx` / `_device` | per-slot + per-ep context | `xhci::SlotCtx`, `EpCtx`, `InputCtx`, `Device` |
| `struct xhci_ring` / `_segment` / `_td` / `_command` / `_virt_ep` | per-ring TRB buffer + per-TD link | `xhci::Ring`, `Segment`, `Td`, `Command`, `VirtEp` |
| `xhci_pci_probe(pdev, id)` | PCIe driver probe | `XhciPci::probe` |
| `xhci_init(hcd)` / `_run(hcd)` / `_stop(hcd)` | per-host lifecycle | `Xhci::init` / `_run` / `_stop` |
| `xhci_alloc_dev(hcd, udev)` / `_free_dev(hcd, udev)` | per-USB-device slot alloc/free | `Xhci::alloc_dev` / `_free_dev` |
| `xhci_setup_addressable_virt_dev(...)` | populate addressed-state slot context | `Xhci::setup_addressable_virt_dev` |
| `xhci_address_device(hcd, udev)` | issue Address Device command | `Xhci::address_device` |
| `xhci_reset_device(hcd, udev)` | issue Reset Device command | `Xhci::reset_device` |
| `xhci_endpoint_init(hcd, virt_dev, udev, ep, mem_flags)` | install endpoint context | `Xhci::endpoint_init` |
| `xhci_check_bandwidth(hcd, udev)` | bandwidth validation | `Xhci::check_bandwidth` |
| `xhci_ring_alloc(xhci, num_segs, cycle_state, type, max_packet, mem_flags)` | alloc TRB ring | `Ring::alloc` |
| `xhci_ring_free(xhci, ring)` | free ring | `Ring::free` |
| `xhci_initialize_ring_info(ring, cycle_state)` | init ring head/tail | `Ring::init_info` |
| `xhci_queue_command(xhci, fields, slot_id, ep_index, cmd_type)` | enqueue command TRB | `Xhci::queue_command` |
| `xhci_queue_bulk_tx(xhci, ...)` / `_intr_tx` / `_isoc_tx` / `_ctrl_tx` | per-type transfer-ring enqueue | `Xhci::queue_*_tx` |
| `xhci_handle_event(xhci, ir)` | per-event-ring dequeue + dispatch | `Xhci::handle_event` |
| `xhci_irq(hcd)` | top-level IRQ handler | `Xhci::irq` |
| `xhci_msi_irq(irq, hcd)` / `_msix_irq(...)` | MSI/MSI-X handlers | `Xhci::msi_irq` / `_msix_irq` |
| `xhci_resume(xhci, msg)` / `_suspend(xhci, do_wakeup)` | system PM | `Xhci::resume` / `_suspend` |
| `inc_enq(xhci, ring, more_trbs_coming)` / `inc_deq(...)` | ring-pointer advance | `Ring::inc_enq` / `_inc_deq` |
| `xhci_ring_ep_doorbell(xhci, slot_id, ep_index, stream_id)` | doorbell write | `Xhci::ring_ep_doorbell` |
| `xhci_ring_cmd_db(xhci)` | command-doorbell write | `Xhci::ring_cmd_db` |
| `xhci_handle_command_completion(xhci, event)` | command-completion dispatch | `Xhci::handle_command_completion` |
| `handle_tx_event(xhci, ir, event)` | transfer-completion dispatch | `Xhci::handle_tx_event` |
| `handle_port_status(xhci, ir, event)` | port-status-change event → root-hub | `Xhci::handle_port_status` |

## Compatibility contract

REQ-1: xHCI 1.2 spec compliance: every documented MMIO register layout + per-context field offset preserved; spec-conformant USB devices enumerate correctly.

REQ-2: PCIe binding via `xhci_pci_probe` matches per-vendor quirks table (Intel Tiger Lake / Alder Lake / Meteor Lake, AMD Promontory / Renoir / Vermeer / Cezanne, ASMedia ASM2142/3142/3242, Etron EJ188, NEC PD720200/202/PD720201, Renesas/uPD720201, VIA VL805/VL806, Fresco FL1100/2000/3000/3500, Texas Instruments TUSB73x0); per-vendor erratum workarounds preserved.

REQ-3: Per-host event-ring + command-ring + per-ep transfer-ring lifecycle: HW-owned cycle-bit semantics correctly observed (consumer's expected cycle-bit toggles every ring-wraparound).

REQ-4: TRB enqueue/dequeue: `inc_enq(more_trbs_coming=true)` for chained-TD-segment doesn't insert Link TRB until end; ring-overflow (no free TRB slots) returns -EBUSY → caller retry-after-doorbell.

REQ-5: Per-slot Device-Context allocation via `xhci_alloc_dev` matches usbcore's `usb_alloc_dev`+`usb_new_device` flow; per-slot context populated via Enable-Slot + Set-Address commands.

REQ-6: Per-endpoint context (`EpCtx`) populated for control + bulk + interrupt + isochronous endpoints; max-packet-size + interval + max-burst + max-streams + max-PStreams from USB descriptor parsed correctly.

REQ-7: USB-3 streams: per-endpoint primary-stream-array allocation supports up to 64K streams (UAS for high-IOPS USB-3 storage); per-stream TRB ring allocated lazily.

REQ-8: USB-3 LPM (U1/U2/U3): per-port U1/U2 timeout negotiation via SetPortFeature(U1_TIMEOUT) / SetPortFeature(U2_TIMEOUT); per-link clkpwr-managed.

REQ-9: DCI (Debug Capability) — xhci-dbgcap chardev for kernel-debugger over USB-C; cross-ref `drivers/usb/host-dbgcap.md` future Tier-3.

REQ-10: Root-hub emulation: `xhci-hub.c` presents the controller's port array as a USB hub to usbcore; per-port status/change/feature operations routed via xHCI commands.

REQ-11: System PM: `xhci_suspend` saves controller state to memory + halts; `xhci_resume` reverses + handles "force quirks" for buggy PCH/SoC integrations that lose state across S3.

REQ-12: Runtime PM (RTPM): per-controller auto-suspend after configurable idle (default 2000ms); per-port D3hot if all attached devices are suspended.

## Acceptance Criteria

- [ ] AC-1: USB-3 NVMe enclosure → SuperSpeed enumeration → IO at expected throughput.
- [ ] AC-2: USB-3 NIC enumerates + IP works.
- [ ] AC-3: USB-2 keyboard + mouse + storage all enumerate via xHCI's USB-2 ports.
- [ ] AC-4: USB-3 LPM test: idle device transitions to U2 within link-timeout; data transfer wakes back to U0.
- [ ] AC-5: xHCI suspend/resume: deep-sleep test; on resume, all devices preserve attached state.
- [ ] AC-6: USB hot-plug stress: 1000x plug/unplug → no urb leak, no devnode leak (KASAN clean).
- [ ] AC-7: USB-3 stream test: UAS-attached enclosure uses streams correctly; verify via `lsusb -t` showing USB Attached SCSI binding.
- [ ] AC-8: kselftest USB tests pass.

## Architecture

`Xhci` lives in `kernel::usb::host::xhci::Xhci`:

```
struct Xhci {
  hcd: Arc<UsbHcd>,
  cap_regs: NonNull<XhciCapRegs>,    // mmio'd
  op_regs: NonNull<XhciOpRegs>,
  run_regs: NonNull<XhciRunRegs>,
  ir_set: NonNull<XhciIntrRegArray>,
  doorbell: NonNull<u32>,             // per-slot+per-ep doorbell array
  cmd_ring: KBox<Ring>,
  event_ring: KBox<EventRing>,
  cmd_list: Mutex<Vec<Arc<Command>>>,
  num_slots: u32,
  num_intrs: u32,
  num_ports: u32,
  port_array: KBox<[Arc<Port>]>,
  devs: Mutex<HashMap<u32, Arc<Device>>>,  // slot_id → Device
  dcbaa: NonNull<u64>,                // Device Context Base Address Array (per-slot ctx pointer)
  scratchpad: KBox<XhciScratchpad>,   // HW-required scratchpad
  ip_caps: u32,
  hci_version: u16,
  hcc_params: u32,
  hcc_params2: u32,
  hcs_params1: u32,
  hcs_params2: u32,
  hcs_params3: u32,
  page_size: u32,
  device_pool: KBox<DmaPool>,
  segment_pool: KBox<DmaPool>,
  small_streams_pool: KBox<DmaPool>,
  medium_streams_pool: KBox<DmaPool>,
  bus_state: [BusState; 2],            // index 0 = USB-2, 1 = USB-3
  shared_hcd: Option<Arc<UsbHcd>>,    // shared USB-2 hcd (legacy)
  pm_state: AtomicU8,
  quirks: u64,
  dbgtty: Option<Arc<DbgTty>>,         // dbgcap UART tty
}
```

`Ring` lives in `kernel::usb::host::xhci::Ring`:

```
struct Ring {
  type_: RingType,                    // CMD / TRANSFER / EVENT / STREAM
  num_segs: u32,
  segment_pool: Arc<DmaPool>,
  first_seg: NonNull<Segment>,
  last_seg: NonNull<Segment>,
  enqueue: NonNull<Trb>,               // enqueue position (kernel writes here)
  enq_seg: NonNull<Segment>,           // enqueue's segment
  dequeue: NonNull<Trb>,               // dequeue position (HW reads from here)
  deq_seg: NonNull<Segment>,
  cycle_state: u8,                     // current expected cycle-bit
  num_trbs_free: u32,
  bounce_buf_len: u32,
  td_list: Mutex<LinkedList<Td>>,
}
```

Probe path `XhciPci::probe`:
1. `pci_enable_device(pdev)` + map MMIO BAR0 (xHCI cap-regs).
2. `usb_create_hcd(hc_driver, dev, ...)` returns a `usb_hcd`; allocate Xhci struct in hcd's hcd_priv.
3. Read cap-regs: `hci_version`, `hcs_params1` (num slots, num intrs, num ports), `hcs_params2` (max scratchpad bufs, ERST max), `hcc_params` (xHCI-supported features).
4. `xhci_init(hcd)`: alloc DCBAA (page-aligned 256-entry pointer array), alloc cmd-ring, alloc event-ring + ERST (Event Ring Segment Table), alloc scratchpad bufs, populate DCBAA[0] = scratchpad-array.
5. `xhci_run(hcd)`: program ERST + DCBAA in regs, set cmd-ring control reg, write Run/Stop bit → HW starts.
6. Register port-array; spawn shared-HCD for USB-2-only endpoints if needed.

Per-USB-device addressing flow:
1. usbcore (`drivers/usb/core/hub.c`) calls `Xhci::alloc_dev(udev)`:
   - Issue Enable-Slot command → wait completion → returns slot_id.
   - Allocate `Device` struct + per-slot input context + output context.
   - Populate slot context: Route String (port-tree path), Speed, Hub flag, Last-Endpoint = 1, Root-Hub Port Number, Number of Ports (if hub).
   - Populate ep0 context: max-packet-size from USB descriptor.
   - DCBAA[slot_id] = device's output-context phys-addr.
2. `Xhci::address_device(udev)`:
   - Issue Address-Device command (input ctx contains slot ctx + ep0 ctx).
   - Wait completion.
   - HW assigns USB device address (1..127); device transitions ADDRESS state.
3. usbcore proceeds with GET_DEVICE_DESCRIPTOR + further enumeration via control transfers on ep0 (which now uses xHCI transfer ring).

Per-transfer enqueue flow (e.g., bulk OUT):
1. usbcore calls `Xhci::queue_bulk_tx(urb)`.
2. Compute number of TRBs needed (1 per per-frag of urb.transfer_buffer + chained-TD).
3. For each TRB:
   - Set `params` = transfer-buffer dma-addr + length (Normal TRB) or per-IDT-bit data.
   - Set `status` = transfer-length + interrupter-target + chain-bit + ENT (Evaluate Next TRB) + ISP (Interrupt-on-Short-Packet) + IOC (Interrupt-on-Completion last TRB only).
   - Set `control` = TRB type (Normal / Setup / Data / Status for control; Normal for bulk; Isoch for isoch; Interrupt for interrupt) + cycle-bit (matches ring's expected) + chain-bit.
   - `Ring::inc_enq(more_trbs_coming = !last_trb)`.
4. Last TRB has cycle-bit toggled from prior ring's expected; HW spots this on poll.
5. `Xhci::ring_ep_doorbell(slot_id, ep_index, stream_id)` writes to MMIO doorbell → HW fetches.
6. HW processes TRBs, posts Transfer Event TRB to event ring on completion.

Event-ring processing `Xhci::handle_event`:
1. Read event TRB at `event_ring.dequeue`.
2. If cycle-bit != ring's expected → no event ready, return.
3. Dispatch by event TRB type:
   - Transfer Event → `Xhci::handle_tx_event(xhci, ir, event)` → look up TD, complete URB.
   - Command Completion → `Xhci::handle_command_completion(xhci, event)`.
   - Port Status Change → `Xhci::handle_port_status(xhci, ir, event)` → root-hub change-bit set.
   - Bandwidth Request, Doorbell Event, Host Controller Event, Device Notification, MFINDEX Wrap → per-spec dispatch.
4. `Ring::inc_deq` advance dequeue; on segment-wrap, toggle cycle-bit.
5. Update HW Event Ring Dequeue Pointer (ERDP) reg.
6. Loop until cycle-bit mismatch.

Per-IRQ handler `Xhci::irq`:
1. Read USBSTS reg to determine if event-ring has work.
2. `Xhci::handle_event(xhci, primary_intr)` to drain event ring.
3. Re-enable interrupt (write USBSTS-EINT bit).

USB-3 LPM: per-port U1/U2 timeout configured via Set Port Feature with U1_TIMEOUT / U2_TIMEOUT; HW auto-transitions per timeout.

System PM: `Xhci::suspend(do_wakeup)`:
1. Halt HW (clear Run bit).
2. Save controller state (cmd-ring + event-ring + DCBAA pointers, ERST, port states) to memory.
3. If do_wakeup: arm wakeup-on-port-change for resume.

`Xhci::resume`: reverse — restore state, set Run bit; for "lost-context" quirk PCHs, fully re-init (skip state-restore).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ring_no_oob` | OOB | TRB enq/deq pointers always within ring's segment chain; segment-wrap correctly handled. |
| `cycle_bit_consistency` | INVARIANT | per-ring cycle-bit matches HW spec across wraparound; consumer never accepts stale TRB. |
| `slot_id_no_oob` | OOB | DCBAA[slot_id] indexed in [1, num_slots]; per-slot Device lookup bounded. |
| `dma_no_uaf` | UAF | per-TRB-ring page lifetime tied to xhci_ring refcount; HW never reads freed page. |

### Layer 2: TLA+

`models/usb/xhci_ring.tla` (parent-declared): proves xHCI command-ring + event-ring + transfer-ring producer-consumer ordering with xHC posting events to ERST while CPU writes commands; cycle-bit + DMS handling correct under concurrent enqueue/dequeue.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Xhci::queue_*_tx` post: `num_trbs_free >= num_trbs_needed`; cycle-bit on emitted TRBs matches `ring.cycle_state` | `Xhci::queue_*_tx` |
| `Xhci::handle_event` post: `event_ring.dequeue` advanced past consumed events; ERDP reg updated atomically | `Xhci::handle_event` |
| `Xhci::address_device` post: per-slot Device's output context's slot-ctx state field == ADDRESSED | `Xhci::address_device` |

### Layer 4: Verus/Creusot functional

`Xhci::queue_bulk_tx(urb) → HW processes TRBs → Xhci::handle_tx_event(event)` round-trip equivalence: completion event matches submitted URB; transfer-length + status fields correct. Encoded as Verus invariant on per-TD lifetime.

## Hardening

(Inherits row-1 features from `drivers/usb/00-overview.md` § Hardening.)

xhci-specific reinforcement:

- **TRB-ring pages allocated from per-NUMA-node DMA pool** for cache locality.
- **Per-context page-aligned alloc** — spec-required; defense against alignment-related HW errata.
- **doorbell write fenced** — every doorbell preceded by mb() to ensure prior TRB writes visible to HW (already true upstream; Rookery makes explicit).
- **Per-segment cycle-bit toggle** — Link TRB at segment end has cycle-bit-toggle-bit set; defense against stale-TRB false-completion.
- **Per-host quirk table audited** — every `XHCI_QUIRK_*` entry has rationale comment + erratum reference; defense against silent quirks.
- **xHCI DCI debug-capability gated** — `xhci-dbgcap` requires CAP_SYS_RAWIO + LSM hook; defense against userspace establishing kernel-debug channel.
- **Per-slot streams alloc bounded** — max-streams capped per-spec (16 PStreams = 64K streams); defense against malicious USB-3 hub claiming insane stream count.
- **PM state-machine audited** — D-state + Run/Halt transitions follow xHCI spec; defense against malformed VMM driving controller into invalid state.
- **Per-port over_current handler rate-limited** — defense against over-current event flood causing scheduler stall.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on every xhci-specific ioctl path (debug-cap chardev `/dev/xhci-dbc*`); per-TRB payload length validated against ring segment size.
- **PAX_KERNEXEC** — W^X on any xHCI firmware/ROM image staged via Renesas/ASMedia firmware-download; debug-cap RX/TX TRB buffers never marked executable.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization at every xhci-dbc ioctl entry.
- **PAX_REFCOUNT** — saturating refcount on `xhci_hcd`, per-slot `xhci_virt_device`, per-endpoint `xhci_virt_ep`, per-ring `xhci_ring`, per-stream `xhci_stream_info`.
- **PAX_MEMORY_SANITIZE** — zero-on-free for DCBAA, ERST, event-ring, command-ring, all device-context pages (input-context contains DMA addrs that must not leak), scratchpad buffers.
- **PAX_UDEREF / PAX_MEMORY_UDEREF** — strict user-pointer access on USBDEVFS-isoc URBs landing in xhci-built isoc TRBs; descriptor pages never user-mappable.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `hc_driver` (xhci variant), per-event-handler dispatch (`handle_cmd_completion`, `handle_port_status`, `handle_tx_event`), `pci_error_handlers`.
- **GRKERNSEC_IO** — xHCI MMIO via `pci_iomap` only; no `iopl/ioperm`; doorbell/runtime/operational register windows bounds-checked against `cap_regs.hccparams`.
- **GRKERNSEC_HIDESYM** — DCBAA physaddr, command-ring physaddr, MSI-X vector ptrs hidden from `/sys/bus/pci/devices/*/xhci_hcd/`.
- **GRKERNSEC_DMESG** — host-system-error (HSE), host-controller-error, port-link-state-error log lines restricted to CAP_SYSLOG.
- **GRKERNSEC_TPE** — Trusted Path Execution for `/dev/xhci-dbc*` debug-capability chardev (gives direct kernel debug pipe).
- **GRKERNSEC_KMOD** — `xhci_pci` / `xhci_plat` auto-bind gated by CAP_SYS_MODULE; vendor-quirk overlay modules signature-required.
- **GRKERNSEC_MODHARDEN** — module signature required for `xhci_hcd`, `xhci_pci`, `xhci_plat_hcd`, `xhci_pci_renesas`.
- **CAP_SYS_RAWIO** strict for `XHCI_DBC_*` debug-capability ioctls (xhci-dbc is a kernel debug channel — gating prevents userspace establishing a covert ring-0 console), Renesas firmware-download path, any `XHCI_QUIRK_*` runtime override.
- **GRKERNSEC_BRUTE** — repeated port-link-state-error / over-current events per port rate-limited; defense against USB-killer / port-storm DoS.
- **PAX-anti-DMA-from-userland** — DCBAA, scratchpad, device-context, and event-ring pages allocated with `dma_alloc_coherent` are not user-mappable; no `mmap()` path exposes them.
- **xhci-dbc + USB4/Thunderbolt PCIe-tunneling** — when xHCI is reached over USB4 tunnel, `iommu_attach_device` must already have isolated the tunnel; grsec policy refuses xhci probe behind a non-isolating ACS path (cross-ref `drivers/iommu/iommu-core.md`).

Per-driver rationale: xHCI is the highest-bandwidth USB controller and uniquely exposes the xhci Debug Capability — a hardware-level kernel-debug channel that, if accessible to unprivileged userspace via `/dev/xhci-dbc*`, would constitute a ring-0 backdoor; additionally USB4/Thunderbolt PCIe-tunneling makes the host xHCI reachable from a downstream PCIe-tunneled device that the user has not explicitly authorized. Grsec reinforces by gating dbgcap behind TPE+CAP_SYS_RAWIO, signing every xhci ops vtable, sanitizing every DMA-coherent context page on free (input-contexts leak slot/endpoint state otherwise), refusing to leak physaddrs of DCBAA/command-ring/event-ring through sysfs, and rate-limiting per-port over-current/PLS-error storms.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- USB-2 EHCI controller (covered in `drivers/usb/host-ehci.md` future Tier-3)
- USB-1.1 OHCI/UHCI (covered in `drivers/usb/host-{ohci,uhci}.md` future Tier-3s)
- Synopsys DesignWare USB3 (covered in `drivers/usb/host-dwc3.md` future Tier-3)
- DesignWare USB2 (covered in `drivers/usb/host-dwc2.md` future Tier-3)
- USB Type-C connector (covered in `drivers/usb/typec-core.md` future Tier-3)
- 32-bit-only paths
- Implementation code
