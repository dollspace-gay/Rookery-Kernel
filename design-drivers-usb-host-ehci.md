---
title: "Tier-3: drivers/usb/host/ehci-*.c — eHCI host controller (USB-2.0 high-speed; QH/qTD periodic-tree + async-list)"
tags: ["tier-3", "drivers-usb", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

eHCI = Enhanced Host Controller Interface — the USB-2.0 high-speed (480 Mbit/s) host controller spec from Intel, supports control/bulk/interrupt/isochronous transfers via a queue-head + queue-element-transfer-descriptor (QH/qTD) two-list model: async-list for control + bulk transfers (linked list of QHs visited round-robin every µframe), periodic-frame-list for interrupt + isochronous transfers (1024 entries indexed by µframe count, each pointing at one or more QHs / iTDs / siTDs scheduled at that µframe). Companion-host model: eHCI handles HS devices; LS/FS devices automatically routed to companion oHCI/uHCI controllers via per-port per-device shadow.

This Tier-3 covers `drivers/usb/host/ehci-*.c` (~7800 lines across 8 files): `ehci-hcd.c` (HCD core), `ehci-hub.c` (root-hub emulation), `ehci-mem.c` (DMA-pool allocators), `ehci-q.c` (async-list QH/qTD), `ehci-sched.c` (periodic-frame-list scheduler — biggest file), `ehci-pci.c` (PCIe binding), `ehci-platform.c` (platform binding for SoC), `ehci-timer.c` (per-controller hrtimers).

### Acceptance Criteria

- [ ] AC-1: USB-2.0 keyboard / mouse / storage / Ethernet adapter enumerate via eHCI on reference HW with eHCI controller.
- [ ] AC-2: USB-2.0 high-speed sustained transfer at near-spec 35 MB/s (flash drive read).
- [ ] AC-3: USB-1.1 device on eHCI port automatically routed to companion controller; works at FS speed.
- [ ] AC-4: USB Audio (HS isochronous) playback via eHCI works without xrun.
- [ ] AC-5: Hot-plug stress: 1000x plug/unplug → no urb leak, no DMA-pool leak (KASAN clean).
- [ ] AC-6: System-suspend / resume preserves attached devices.
- [ ] AC-7: kselftest USB-2 EHCI subset passes.

### Architecture

`Ehci` lives in `kernel::usb::host::ehci::Ehci`:

```
struct Ehci {
  hcd: Arc<UsbHcd>,
  caps: NonNull<EhciCapsRegs>,         // mmio'd
  regs: NonNull<EhciOpRegs>,
  hcs_params: u32,
  hcc_params: u32,
  command: u32,                          // USBCMD cache
  status: AtomicU32,                     // last USBSTS read
  async_: AtomicPtr<Qh>,                 // async-list head (sentinel QH)
  async_unlink: Mutex<Vec<Arc<Qh>>>,     // QHs awaiting IAA-driven unlink
  intr_unlink: Mutex<Vec<Arc<Qh>>>,
  iaa_in_progress: AtomicBool,
  periodic: NonNull<u32>,                // 1024-entry periodic-frame-list (DMA-coherent)
  periodic_dma: dma_addr_t,
  pshadow: KBox<[PeriodicShadow; 1024]>, // SW-shadow of periodic for fast lookup
  next_uframe: AtomicU32,
  random_frame: AtomicU32,
  last_periodic_enable: AtomicU64,
  qh_pool: Arc<DmaPool>,                  // per-controller QH alloc pool
  qtd_pool: Arc<DmaPool>,
  itd_pool: Arc<DmaPool>,
  sitd_pool: Arc<DmaPool>,
  hrtimer: HrTimer,
  enabled_hrtimer_events: u32,
  rh_state: RhState,                      // root-hub state
  bus_state: [BusState; 1],
  ...
}

struct Qh {
  hw: NonNull<EhciQhHw>,                  // HW-visible 32 bytes (DMA-coherent)
  qh_dma: dma_addr_t,
  qtd_list: ListHead<Qtd>,
  qh_next: NextLink,                      // linked into async or periodic
  ep: Arc<UsbHostEndpoint>,
  refcount: Refcount,
  qh_state: QhState,
  ...
}
```

Probe path `EhciPci::probe`:
1. PCI enable + memremap MMIO BAR0.
2. `usb_create_hcd(...)` → allocate `usb_hcd`, embed Ehci.
3. `Ehci::setup(hcd)`: read CAPLENGTH/HCSPARAMS/HCCPARAMS; enable async + periodic schedules.
4. Allocate periodic-frame-list (4 KB, DMA-coherent); init sentinel QH for async; init DMA-pools (QH/qTD/iTD/sITD).
5. `Ehci::run(hcd)`: configure registers (PERIODICLISTBASE + ASYNCLISTADDR + USBCMD.RUN); set CONFIGFLAG=1 to claim ports.

Async-list submit `Ehci::urb_enqueue` (control/bulk):
1. Look up or alloc per-endpoint QH.
2. For each transfer-buffer chunk (PRP-style up to 5 4K pages per qTD):
   - Allocate qTD from qtd_pool.
   - Populate token (PID + length + IOC if last + DT) + buffer-page-pointers.
3. Link qTDs via next-pointer chain.
4. Append qTD-list to QH's qtd_list.
5. If QH not on async-list: `qh_link_async(ehci, qh)`.

Periodic-list submit (interrupt URB):
1. Look up or alloc per-endpoint QH.
2. Compute per-µframe slot mask via `qh_schedule(...)` — bandwidth-allocator finds free µframes per HS interval.
3. For each `frame_index` in periodic-list matching schedule:
   - If pshadow[frame_index] empty: install QH directly.
   - Else: link QH via per-frame chain (sorted by polling rate).
4. Set qh.endpoint-capabilities.s-mask + c-mask per scheduled µframes.

Iso URB submit (HS isoch):
1. Per-stream `IsoStream` lookup or alloc.
2. Per-URB allocate iTD list (one per 8 µframes).
3. Per-iTD: populate transaction[8] entries + buffer-pointer with per-microframe data offsets.
4. Insert iTDs into periodic-list at scheduled µframe slot.

IRQ handler `Ehci::irq`:
1. Read USBSTS, ack consumed bits.
2. If INT or ERR: scan async-list + periodic-list for completed qTDs/iTDs; per-QH `qh_completions(ehci, qh)` → walk qTDs, if status field shows complete: usb_hcd_giveback_urb (cross-ref `core-urb.md`).
3. If PCD: walk PORTSC[]; trigger hub_event for changed ports.
4. If IAA: process pending qh_unlink list (HW has advanced past, safe to unlink).
5. If HSE: halt controller + emit warning.

Companion-controller hand-off: per-port `PORT_OWNER` bit; when PORT_CONNECTION + LS/FS device detected, set PORT_OWNER → companion oHCI/uHCI takes over the port.

System PM: `ehci_pci_suspend` saves controller state to memory + halts; `ehci_pci_resume` restores or re-runs init for "lost-state" PCH.

### Out of Scope

- xHCI (covered in `drivers/usb/host-xhci.md` Tier-3)
- oHCI / uHCI companion-controllers (covered in `drivers/usb/host-{ohci,uhci}.md` future Tier-3s)
- DesignWare USB2 (covered in `drivers/usb/host-dwc2.md` future Tier-3)
- USB Type-C connector (covered in `drivers/usb/typec-core.md` future Tier-3)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ehci_hcd` | per-controller control block | `kernel::usb::host::ehci::Ehci` |
| `struct ehci_qh` | per-endpoint queue-head | `kernel::usb::host::ehci::Qh` |
| `struct ehci_qtd` | per-transfer queue-element-transfer-descriptor | `kernel::usb::host::ehci::Qtd` |
| `struct ehci_itd` | per-isoch HS transfer descriptor | `kernel::usb::host::ehci::Itd` |
| `struct ehci_sitd` | per-isoch FS-via-companion split-transfer descriptor | `kernel::usb::host::ehci::Sitd` |
| `struct ehci_caps` / `_regs` | per-controller MMIO register layout | `kernel::usb::host::ehci::regs::*` |
| `ehci_pci_setup(hcd)` | PCIe binding setup | `EhciPci::setup` |
| `ehci_setup(hcd)` | per-controller init | `Ehci::setup` |
| `ehci_run(hcd)` | start controller | `Ehci::run` |
| `ehci_stop(hcd)` | halt controller | `Ehci::stop` |
| `ehci_urb_enqueue(hcd, urb, mem_flags)` | per-URB submit | `Ehci::urb_enqueue` |
| `ehci_urb_dequeue(hcd, urb, status)` | per-URB cancel | `Ehci::urb_dequeue` |
| `qh_link_async(ehci, qh)` | append QH to async-list | `Qh::link_async` |
| `qh_unlink_async(ehci, qh)` | unlink QH from async-list (deferred via IAA interrupt) | `Qh::unlink_async` |
| `qh_alloc(ehci, flags)` / `qh_destroy(ehci, qh)` | QH alloc/free from per-controller DMA pool | `Qh::alloc` / `_destroy` |
| `qtd_alloc(ehci, flags)` / `qtd_free(ehci, qtd)` | qTD alloc/free | `Qtd::alloc` / `_free` |
| `qh_completions(ehci, qh)` | per-QH completion processing | `Qh::process_completions` |
| `intr_submit(ehci, urb, &qtd_list, mem_flags)` | interrupt URB submit (periodic-list) | `Ehci::intr_submit` |
| `iso_stream_alloc(...)` / `_free(...)` | per-iso-endpoint stream | `IsoStream::alloc` / `_free` |
| `iso_sched_alloc(...)` / `_free(...)` | per-iso-URB schedule | `IsoSched::alloc` / `_free` |
| `itd_link_urb(ehci, urb, ...)` / `sitd_link_urb(...)` | iso descriptor link | `Itd::link_urb` / `Sitd::link_urb` |
| `scan_isoc(ehci)` | per-µframe isoch completion scan | `Ehci::scan_isoc` |
| `ehci_hub_status_data(hcd, buf)` | per-port status report | `Ehci::hub_status_data` |
| `ehci_hub_control(hcd, typeReq, value, index, buf, length)` | hub-control request dispatch | `Ehci::hub_control` |
| `ehci_irq(hcd)` | top-level IRQ handler | `Ehci::irq` |
| `ehci_endpoint_disable(hcd, ep)` | per-endpoint cleanup | `Ehci::endpoint_disable` |
| `ehci_endpoint_reset(hcd, ep)` | per-endpoint toggle reset | `Ehci::endpoint_reset` |
| `ehci_get_frame(hcd)` | get current µframe count | `Ehci::get_frame` |

### compatibility contract

REQ-1: eHCI 1.0 spec compliance: per-controller MMIO register layout (CAPLENGTH / HCSPARAMS / HCCPARAMS / USBCMD / USBSTS / USBINTR / FRINDEX / CTRLDSSEGMENT / PERIODICLISTBASE / ASYNCLISTADDR / CONFIGFLAG / PORTSC[N]) byte-identical.

REQ-2: PCIe binding via per-vendor quirks: Intel ICH/PCH (eHCI absorbed into PCH then later xHCI; older Sandy Bridge / Lynx Point / Wildcat Point still have eHCI), AMD chipsets (older HudsonD / Bolton), ASMedia, NVIDIA MCP-series, VIA, SiS — all per-PCI-id table.

REQ-3: Per-QH structure per spec: HW-visible portion (32 bytes: link-pointer + endpoint-characteristics + endpoint-capabilities + current-qTD-pointer + overlay-area), SW-only portion (per-driver state).

REQ-4: Per-qTD structure per spec: 32 bytes (next-pointer + alt-next-pointer + token + 5-page-buffer pointers).

REQ-5: Async-list (control + bulk): circular linked list of QHs visited HW round-robin every µframe; each QH walked qTD-by-qTD until empty or NAK.

REQ-6: Periodic-frame-list (interrupt + isoch): 1024 entries (indexed by FRINDEX[12:3] = current µframe); each entry points at one or more QHs/iTDs/sITDs to be processed at this µframe.

REQ-7: HS interrupt URB: scheduled by inserting QH into per-µframe periodic list slots; `qh.endpoint-capabilities.s-mask + c-mask` determines start + complete bits per µframe.

REQ-8: HS isochronous URB: per-iTD (8 microframes per iTD); per-stream IsoStream object aggregating per-frame iTDs.

REQ-9: FS-via-companion split-transfer: when HS hub bridges FS device, controller sends start-split/complete-split tokens; per-sITD records FS-data-transfer + start/complete µframe scheduling.

REQ-10: Root-hub emulation: per-port USB-2.0 status reported via PORTSC register; companion-controller hand-off via PORT_OWNER bit (LS/FS devices auto-routed to oHCI/uHCI).

REQ-11: Per-controller IRQ handler `ehci_irq`: drains USBSTS bits — INT (qTD completion), ERR (transaction error), PCD (port-change-detect → wake hub_event), FLR (frame-list-rollover → re-queue periodic), HSE (host-system-error → halt + reset), IAA (Interrupt-on-Async-Advance, used to defer QH unlink until HW has actually advanced past).

REQ-12: System PM: `ehci_suspend` saves controller state + halts; `ehci_resume` restores. Per-vendor "force-quirk" for chipsets that lose state across S3.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `qh_no_uaf` | UAF | `Arc<Qh>` outlives all in-flight qTDs; release waits for IAA-deferred unlink. |
| `qtd_pool_no_oob` | OOB | per-pool DMA-pool alloc bounded by pool capacity; over-alloc returns -ENOMEM. |
| `periodic_idx_no_oob` | OOB | periodic-list index bounded to [0, 1024). |
| `bandwidth_alloc_consistent` | INVARIANT | per-µframe bandwidth allocation never exceeds 60% (high-speed budget reserve for unscheduled). |

### Layer 2: TLA+

(eHCI shares `models/usb/urb_lifetime.tla` from parent — no eHCI-specific TLA+ model.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ehci::urb_enqueue` post: URB queued on appropriate (async or periodic) list AND completion will be called exactly once | `Ehci::urb_enqueue` |
| `Qh::process_completions` invariant: per-qTD status-field examined exactly once; on complete, qTD removed from list + URB given back | `Qh::process_completions` |
| `Ehci::scan_isoc` invariant: per-iTD per-microframe status examined; URB given back at last µframe completion | `Ehci::scan_isoc` |
| Per-QH state-machine: IDLE → LINKED (async or periodic) → UNLINK_WAIT → UNLINKED → IDLE; valid transitions only | `Qh::state` |

### Layer 4: Verus/Creusot functional

`Ehci::urb_enqueue → HW DMA → qh_completions → urb.complete` round-trip equivalence: completion's URB has correct `actual_length` + `status` reflecting HW state for any successful transfer.

### hardening

(Inherits row-1 features from `drivers/usb/00-overview.md` § Hardening.)

eHCI-specific reinforcement:

- **Per-QH/qTD/iTD/sITD allocated from per-controller DMA-pool** for deterministic alignment + no fragmentation.
- **Periodic-list size fixed at 1024 entries** per spec; defense against userspace-supplied URB requesting bandwidth beyond schedule capacity.
- **HSE (Host-System-Error) handler halts controller** + per-PCI-recovery; defense against HW bug causing system DMA corruption.
- **IAA-deferred QH unlink** ensures HW has advanced past the QH's last position before SW frees memory; defense against HW dereferencing freed QH.
- **Companion-controller PORT_OWNER per-port hand-off atomic** — defense against per-port owner-state-confusion.
- **Per-controller pm-state save/restore audited** — every register restored matches saved value (validated post-resume).
- **DMA-pool descriptor zeroed before reuse** — defense against stale-data leak across URBs.
- **Per-vendor quirk table audited** — every entry has rationale comment + erratum reference.

