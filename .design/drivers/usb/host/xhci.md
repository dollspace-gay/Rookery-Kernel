# Tier-3: drivers/usb/host/{xhci,xhci-ring,xhci-mem,xhci-hub,xhci-pci,xhci-plat,xhci-dbgcap,xhci-dbgtty,xhci-debugfs,xhci-sideband,xhci-ext-caps}.c — Generic xHCI USB 3.x host controller (every modern motherboard, every Raspberry Pi 4/5, every laptop)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/usb/00-overview.md
upstream-paths:
  - drivers/usb/host/xhci.c                          (~5720 lines: hcd_driver ops, controller init, suspend/resume, watchdog, USB roothub)
  - drivers/usb/host/xhci.h                          (~2400 lines: types, register layout, TRB definitions)
  - drivers/usb/host/xhci-ring.c                     (~4490 lines: TRB ring management, command/event/transfer rings, doorbell, ring dequeue update)
  - drivers/usb/host/xhci-mem.c                      (~2510 lines: device context array, EP context, scratchpad, DCBAA, segment table, alloc/free)
  - drivers/usb/host/xhci-hub.c                      (~1970 lines: USB hub emulation — port-status, reset, suspend, BESL/U1/U2/U3 power states, USB3 LPM)
  - drivers/usb/host/xhci-pci.c                      (~985 lines: PCI bus glue + vendor quirks for Intel/AMD/Renesas/ASMedia/Etron etc.)
  - drivers/usb/host/xhci-pci-renesas.c              (~665 lines: Renesas uPD720201/720202 firmware download)
  - drivers/usb/host/xhci-plat.c                     (~660 lines: platform bus glue for SoCs)
  - drivers/usb/host/xhci-dbgcap.c                   (~1560 lines: DbC — USB3 debug capability, kernel-debug-over-USB)
  - drivers/usb/host/xhci-dbgtty.c                   (~670 lines: DbC tty wrapper /dev/ttyDBC*)
  - drivers/usb/host/xhci-debugfs.c                  (~855 lines: debugfs surface)
  - drivers/usb/host/xhci-sideband.c                 (~495 lines: sideband interface for audio offload)
  - drivers/usb/host/xhci-ext-caps.c                 (extended caps walker)
  - drivers/usb/host/xhci-mtk.c + xhci-mtk-sch.c     (MediaTek SoC variant — BW scheduler)
  - drivers/usb/host/xhci-tegra.c                    (NVIDIA Tegra variant — partner with FW)
  - drivers/usb/host/xhci-histb.c, xhci-rcar.c, xhci-mvebu.c, xhci-rzv2m.c  (other SoC variants)
  - drivers/usb/host/xhci-caps.h
  - include/linux/usb/{ch9,hcd,xhci-sideband}.h
-->

## Summary

xHCI (eXtensible Host Controller Interface, Intel spec rev 1.2a) is the USB 3.x host controller standard — the silicon every USB 3 port on every modern computer is wired to. The generic xHCI driver in `drivers/usb/host/xhci.c` is the bus-agnostic core; PCI glue (`xhci-pci.c`) and platform-device glue (`xhci-plat.c`) wrap it for x86 + SoC integration. Vendor-specific quirks for ~30 distinct PCI vendor:device pairs (Intel ICH/PCH/Skylake/IceLake/AlderLake, AMD Bolton/Raven, Renesas uPD720201/720202, ASMedia ASM1042/1142/2142/3242, Etron, Fresco, VIA) live in `xhci-pci.c`. SoC variants (MediaTek MT76xx with hardware bandwidth scheduler, NVIDIA Tegra with FW-mediated XUSB partition, HiSilicon, Renesas R-Car) live in per-SoC files.

xHCI replaced both EHCI (USB 2.0) and OHCI/UHCI (USB 1.1) for any USB 3 port; on most x86 systems, USB 2.0 devices also enumerate via xHCI through the Rate-Matching Hub built into the controller silicon (`xhci_check_pause` and roothub emulation handle this). USB 2.0-only ports may still use EHCI on older platforms.

This Tier-3 covers ~26300 lines of generic + PCI + DbC + SoC xHCI code.

## Rust translation posture

xHCI is one of the highest-CVE-density drivers in Linux — multiple memory corruption + race CVEs over the years (CVE-2024-26663 race in ring-management, CVE-2023-1192 use-after-free in slot-disable, CVE-2018-1093 OOB read in xhci-dbg). The ring-management code (`xhci-ring.c`) is the highest-bug-density file. Rust translation:

- **TRB rings as typed.** Each ring type (command, event, transfer) is a distinct type: `CommandRing`, `EventRing`, `TransferRing<EP>`. The TRB cycle bit + chain bit semantics are encoded as type-level invariants — a TRB cannot be enqueued without the right cycle bit by construction.
- **Ring dequeue update + doorbell** as an atomic operation; closes a known race class.
- **DCBAA (Device Context Base Address Array)** typed: `Dcbaa = [DeviceContextPtr; MAX_DEVICES]`; per-slot context is `DeviceContext<S>` where `S` is the slot state (`Disabled`, `Default`, `Addressed`, `Configured`).
- **EP context state machine.** USB EP states (`Disabled`, `Running`, `Halted`, `Stopped`, `Error`) typed; methods to change state are gated.
- **Roothub emulation.** xHCI driver presents a virtual hub to the USB core; the per-port state machine (DISCONNECTED / ENABLED / SUSPENDED / RESETTING) typed.
- **USB3 LPM (Link Power Management).** U0/U1/U2/U3 link states modelled as typed transitions.
- **DbC (Debug Capability)** — kernel-debug-over-USB. Sensitive: a debugger attached via DbC has full kernel memory access. Strict CAP_SYS_ADMIN + lockdown integration.
- **Vendor quirks** — typed `XhciQuirks` bitset; per-vendor matched in pci-bus-glue.

The grsec/PaX section is mandatory: xHCI is the cross-privilege-boundary between userspace USB device drivers and host kernel memory; DMA from a malicious USB device can target arbitrary host memory if IOMMU is not enforcing. BadUSB-style attacks live here.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct xhci_hcd` | per-HC state (rings, DCBAA, device-slot array, regs) | `XhciHcd<Phase>` |
| `struct xhci_virt_device` | per-USB-device state (slot id, EP rings) | `VirtDevice<S>` |
| `struct xhci_virt_ep` | per-EP state (ring, EP context, ep_state) | `VirtEp<EpState>` |
| `struct xhci_ring` | TRB ring + cycle state + first/last seg | `Ring<RingType>` |
| `struct xhci_segment` | TRB segment (256 TRBs typically) | `Segment` |
| `struct xhci_command` | in-flight command (queued on cmd ring) | `Command<Status>` |
| `xhci_pci_probe(...)` / `_remove(...)` | PCI probe / remove | `XhciHcd::probe_pci` / `Drop` |
| `xhci_plat_probe(...)` | platform probe (SoC) | `XhciHcd::probe_plat` |
| `xhci_init(hcd)` / `xhci_run(hcd)` / `xhci_stop(hcd)` | HCD lifecycle | `XhciHcd::init` / `run` / `stop` |
| `xhci_handle_event(xhci)` / `_handle_command_completion` / `_handle_tx_event` | event ring handler | `EventRing::handle` |
| `xhci_queue_command(xhci, cmd, slot_id, ...)` / `_queue_address_device(...)` / `_queue_configure_endpoint(...)` / `_queue_stop_endpoint(...)` / `_queue_reset_endpoint(...)` | cmd issue | `CommandRing::queue_*` |
| `xhci_ring_cmd_db(xhci)` / `_ring_doorbell_for_active_rings(...)` | doorbell | `Doorbell::ring_cmd` / `_xfer` |
| `xhci_alloc_dev(hcd, udev)` / `_free_dev(...)` | per-USB-device alloc / free | `VirtDevice::alloc` / `Drop` |
| `xhci_alloc_command(xhci, allocate_completion, mem_flags)` / `_free_command(...)` | command alloc / free | `Command::alloc` / `Drop` |
| `xhci_setup_addressable_virt_dev(...)` / `_setup_no_data_command(...)` | EP/device context setup | `VirtDevice::setup_*` |
| `xhci_endpoint_init(...)` / `_endpoint_zero_bandwidth(...)` | EP init | `VirtEp::init` / `_zero_bw` |
| `xhci_add_endpoint(hcd, udev, ep)` / `_drop_endpoint(...)` / `_check_bandwidth(...)` / `_reset_bandwidth(...)` | EP add/drop/check | `VirtDevice::add_ep` / etc. |
| `xhci_alloc_streams(...)` / `_free_streams(...)` | bulk streams alloc | `VirtEp::alloc_streams` |
| `xhci_urb_enqueue(hcd, urb, mem_flags)` / `_urb_dequeue(hcd, urb, status)` | URB submit / cancel | `XhciHcd::enqueue_urb` / `_dequeue_urb` |
| `xhci_queue_intr_tx(...)` / `_queue_bulk_tx(...)` / `_queue_ctrl_tx(...)` / `_queue_isoc_tx_prepare(...)` | TX per-endpoint-type | `TransferRing::queue_*` |
| `xhci_hub_control(hcd, type_req, value, index, buf, length)` / `_hub_status_data(...)` | roothub emulation | `RootHub::control` / `_status` |
| `xhci_bus_suspend(hcd)` / `_bus_resume(...)` | bus suspend/resume | `Bus::suspend` / `resume` |
| `xhci_dbc_init()` / `_exit()` / `_dbc_handle_events(work)` | DbC | `Dbc::init` / `_exit` / `_handle_events` |
| `xhci_handle_command_completion(...)` | cmd completion dispatch | `CommandRing::on_completion` |
| `handle_set_deq_completion(...)` / `_set_dequeue_pointer_completion(...)` | dequeue ptr update | `Ring::on_set_deq` |
| `xhci_stop_endpoint_command_watchdog(timer)` | stop-EP watchdog | `Watchdog::stop_ep` |
| `xhci_set_link_state(xhci, port, link_state)` | USB3 link state change | `Port::set_link_state` |
| `compliance_mode_recovery` / `_recovery_timer_init(...)` | USB3 compliance-mode workaround | `ComplianceMode::recover` |

## Compatibility contract

REQ-1: xHCI spec rev 1.2a — driver matches the published Intel spec. Capability registers + operational registers + runtime registers + doorbell array all at spec-defined offsets.

REQ-2: Per-HC resource layout — CRCR (Command Ring Control), DCBAA (Device Context Base Address Array, indexed by slot_id), ERST (Event Ring Segment Table), per-slot Device Context + per-EP EP Context, scratchpad buffers (FW-side temp memory). All DMA-mapped with appropriate alignment per spec.

REQ-3: TRB ring discipline — each ring has a producer cycle state (PCS) tracked by the driver and a consumer cycle state (CCS) tracked by the controller. TRBs are valid when their cycle bit equals PCS at write time; controller advances by consuming TRBs whose cycle bit matches its CCS. Wrap via Link TRB (toggles cycle bit). Frozen against xHCI spec.

REQ-4: Slot lifecycle — `Disabled → Default → Addressed → Configured`. ENABLE_SLOT → ADDRESS_DEVICE → CONFIGURE_ENDPOINT. Deinit reverses: DISABLE_SLOT.

REQ-5: EP lifecycle — `Disabled → Running → Halted/Stopped → Error → Disabled`. STOP_ENDPOINT pauses transfers, RESET_ENDPOINT clears halt, CONFIGURE_ENDPOINT modifies the EP context.

REQ-6: URB submit/complete — URB enqueue writes TRBs into the EP's transfer ring + rings the doorbell; URB complete on a Transfer Event TRB on the event ring.

REQ-7: USB hub emulation — driver presents a virtual hub to USB core. Hub descriptor synthesized from `MAX_PORTS` register. Port status (CSC/PEC/PRC/etc.) translated from xHCI port-status register format.

REQ-8: USB3 LPM (U1/U2 entry/exit) — handled per spec; BESL (Best Effort Service Latency) values negotiated per device.

REQ-9: USB3 hot-plug — port-status change events on the event ring trigger USB-core notifications.

REQ-10: Bulk streams — Up to 64K streams per EP for USB Attached SCSI (UAS) etc.

REQ-11: Isochronous transfers — frame-id scheduling per spec; missed-frame handling.

REQ-12: Suspend/resume — D3 entry quiesces, D0 restore re-arms; with vendor quirks for chips that don't preserve state through D3.

REQ-13: DbC (Debug Capability) — when CONFIG_USB_XHCI_DBGCAP=y and the controller advertises DbC ext-cap, a USB3 debug device is enumerated on the host; `/dev/ttyDBC0` exposes a tty for kernel-debug-over-USB.

REQ-14: SoC variants — per-SoC files for MTK (BW scheduler), Tegra (FW-mediated), HiSilicon, R-Car. Each registers via xhci-plat with vendor-specific init.

REQ-15: PCI vendor quirks — per-vendor `xhci_pci_quirks[]`; >30 distinct vendor:device pairs. Examples: ASMedia ASM1142 short-packet quirk, Renesas FW download required at probe, Intel skylake reset workaround.

## Acceptance Criteria

- [ ] AC-1: `lspci -d ::0c0330` shows xHCI controllers; Rookery probes; `dmesg | grep xhci` matches upstream init log.
- [ ] AC-2: USB 3 thumb drive plugged in: enumerates as USB 3 device; `lsusb -v` shows correct descriptors; mounts and reads/writes at SuperSpeed (≥400 MB/s on USB 3.0, ≥1 GB/s on USB 3.1 Gen 2).
- [ ] AC-3: USB 2.0 device plugged into USB 3 port: enumerates through the Rate-Matching Hub at HighSpeed (480 Mb/s).
- [ ] AC-4: USB hub with multiple devices: enumeration cascades correctly; all devices reachable.
- [ ] AC-5: Bulk streams (UAS): USB 3 NVMe enclosure presents as `/dev/sdN`; uses bulk streams; fio sustained 1 GB/s read.
- [ ] AC-6: Isoc transfers: USB webcam captures at full frame rate; no dropped frames; `v4l2-ctl --stream-mmap` works.
- [ ] AC-7: Hot plug: 1000 plug/unplug cycles on a single port; no leaked endpoints; no kernel splat.
- [ ] AC-8: USB3 LPM: `echo "auto" > /sys/.../power/control`; U1/U2 entry/exit observed via debugfs trace; latency increase within spec.
- [ ] AC-9: Bus suspend/resume: system-suspend (S3) + resume; all devices re-enumerate cleanly.
- [ ] AC-10: DbC: with CONFIG_USB_XHCI_DBGCAP enabled, connect to another Linux host via USB 3 cable; `/dev/ttyDBC0` opens; `picocom` works as a serial console.
- [ ] AC-11: Vendor quirks: Renesas uPD720202 (which needs FW download at probe): driver loads FW from `/lib/firmware/renesas_usb_fw.mem`; controller comes alive.

## Architecture

**Ring discipline.** xHCI has three ring kinds:
- **Command Ring** — one per HC. Driver writes commands (ENABLE_SLOT, ADDRESS_DEVICE, etc.); FW reads. Single cycle bit tracked by driver.
- **Event Ring** — one per interrupter (IRQ vector). FW writes events; driver reads.
- **Transfer Ring** — one per EP per device. Driver writes Transfer TRBs; FW reads, executes USB transactions, writes Transfer Event TRBs back on the event ring.

Each ring is a linked list of `xhci_segment`s (typically 256 TRBs each). The last TRB of each segment is a Link TRB that points to the next segment (and toggles the cycle bit on wrap). Driver tracks `enq` (enqueue pointer) and `deq` (dequeue pointer); FW tracks its own dequeue position.

Rookery's `Ring<RingType>` carries the type parameter; methods are gated. `CommandRing::queue` returns a `Command<Pending>` handle that the caller awaits; on completion the handle becomes `Command<Done>` or `Command<Failed>`.

**Slot + EP lifecycle.** Per-USB-device:
1. `ENABLE_SLOT` cmd → FW returns slot_id (1..max_slots).
2. Driver fills the Device Context at DCBAA[slot_id] (input ctx) + EP0 Context.
3. `ADDRESS_DEVICE` cmd → FW writes USB address.
4. `CONFIGURE_ENDPOINT` cmd → FW activates additional EPs.

USB-core then issues URBs; driver writes Transfer TRBs to the corresponding EP's transfer ring; FW executes; event TRBs come back; URB completed.

Deinit: `STOP_ENDPOINT` per EP → `DISABLE_SLOT` → free DCBAA entry.

**EP context state machine.** USB EP states `Disabled / Running / Halted / Stopped / Error`. STOP_ENDPOINT pauses. RESET_ENDPOINT clears halt. The CONFIGURE_ENDPOINT cmd with input-control flags modifies the EP without bouncing the whole device. Rookery's typed `VirtEp<EpState>` gates which ops are valid per-state.

**URB enqueue → TRB.** A bulk URB with N fragments lowers to N Normal TRBs (with chain bit set on all but the last) + an Event Data TRB at the end (signals completion with the URB's user_data). The driver writes the TRBs into the EP's transfer ring, increments enq, rings the per-EP doorbell. Cycle bit handling: each TRB written with the current PCS; on segment wrap, PCS flips.

**Event handling.** IRQ → event ring poll → for each TRB type:
- `Command Completion Event` → look up the cmd by `Command TRB Pointer`, set its status, wake waiter.
- `Transfer Event` → look up the EP, walk back from the EP's transfer ring deq to find the URB that owns this TRB, complete the URB.
- `Port Status Change Event` → notify USB core of port change.
- `Doorbell Event` (rare) → debug.

The "walk back to find URB" step is a known hot spot for ring bugs (CVE-2024-26663 and others). Rookery's typed ring carries explicit URB-trb-list associations.

**Roothub emulation.** xHCI doesn't have a separate USB-hub silicon — the controller's USB ports are the roothub. Driver presents a synthetic hub descriptor; port-status (CSC, PEC, PRC, OCC, PRC, WRC, ...) is translated from xHCI PORTSC register format to USB hub-spec format. `hub_control` and `hub_status_data` handlers implement the standard hub-class requests.

**DbC.** When the chip advertises DbC ext-cap, a kernel-debug-over-USB capability becomes available. Driver registers `/dev/ttyDBC0`; data written there flows over the USB 3 cable to a host that's running a DbC consumer (typically another Linux machine running `udevadm monitor` or `picocom`). DbC is a privileged surface — gating below.

**Vendor quirks.** `xhci-pci.c` matches PCI vendor:device and sets `xhci->quirks` bits like `XHCI_RESET_ON_RESUME`, `XHCI_AMD_PLL_FIX`, `XHCI_BROKEN_PORT_PED`, etc. Quirks gate behavior in the generic code via `if (xhci->quirks & XHCI_*)`.

## Hardening

- TRB segment alignment validated at alloc (xHCI spec mandates page-aligned segments).
- Per-EP transfer ring lookup bounds-checked; an event with an out-of-range EP index is dropped.
- Slot_id from FW validated against `max_slots` cap before DCBAA access.
- Command completion's `Command TRB Pointer` validated against the command ring's address range.
- All FW-supplied fields in events validated against expected ranges before use.
- URB cancellation handles in-flight-stop-EP race correctly (the historical CVE source).
- Vendor-quirk gating handled at probe; runtime quirk change refused.
- Roothub-control payload validated against expected size for each control request type.
- Bulk streams alloc/free reference-counted; no leak across config change.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — DbC `/dev/ttyDBC*` read/write uses bounded `copy_to/from_user`; sysfs port-control uses bounded reads. URB transfer buffers come from USB-driver layer (already bounded there).
- **PAX_KERNEXEC** — `xhci_hc_driver`, `xhci_pci_quirks[]`, vendor-PCI match table, hub-emulation handlers, DbC ops all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — sysfs port-power, debugfs entry paths are per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `xhci_hcd` refcount, per-`xhci_virt_device` refcount, per-`xhci_command` refcount, per-DbC-device refcount use saturating refcounts; underflow at remove is a hard panic.
- **PAX_MEMORY_SANITIZE** — TRB ring segments cleared on free (may contain HID/keyboard scancodes, smartcard data, USB audio audio data). Per-EP context cleared on free. DbC buffers cleared on close.
- **PAX_UDEREF** — SMAP/SMEP on DbC tty, debugfs, sysfs.
- **PAX_RAP / kCFI** — `xhci_hc_driver` ops, IRQ handler dispatch, event TRB handler table, hub-emulation handlers, DbC notifier all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/usb/xhci/*`) dumps include ring segment addresses, EP context pointers, DCBAA pointers; non-root reads scrub addresses.
- **GRKERNSEC_DMESG** — xHCI verbose logs (TRB-by-TRB trace, slot/EP state transitions) restricted from non-root dmesg; can leak per-device traffic patterns.
- **CAP_SYS_RAWIO required on DbC tty** — `/dev/ttyDBC*` open requires CAP_SYS_RAWIO + LOCKDOWN_DEBUGFS. DbC is a kernel-debug surface; a process with DbC access has equivalent of /dev/kmem read+write (over the USB cable).
- **DbC strict lockdown integration** — when LOCKDOWN_KERNEL_RAM is active, DbC is fully disabled. DbC + an unprivileged attacker with physical USB access = ring 0 access; the lockdown gate is mandatory.
- **IOMMU mandatory for xHCI DMA** — xHCI does scatter-gather DMA across the bus. Without an IOMMU enforcing per-device DMA permissions, a malicious USB device or a buggy/compromised xHCI controller can DMA to arbitrary host memory (the "Thunderclap" / "FireWire-DMA" attack class extended to USB). Rookery refuses to probe an xHCI controller unless the IOMMU group reports per-device isolation.
- **TRB event payload validation strict** — every event TRB's slot_id, EP index, TRB pointer validated against driver-owned ranges before dereferencing. Closes the "compromised xHCI controller forges event with OOB pointer" class.
- **Stop-EP / Cancel-URB race fixed structurally** — Rookery's typed VirtEp ensures STOP_ENDPOINT and URB dequeue cannot race; the EP state transitions through `Stopping → Stopped` typed-state and URB dequeue only on `Stopped`. Closes CVE-2024-26663 class.
- **Roothub-control request size validated** — every hub-class control request validated for size + setup-packet format before parsing. Closes parser-OOB class.
- **Vendor-quirk table immutable after init** — quirk bits set at probe, never modified. Refuses runtime quirk-change requests via debugfs.
- **USB Type-C role swap gated** — role-swap ops require CAP_SYS_ADMIN.
- **BIOS-handoff (extended capability) bounded** — the BIOS-to-OS handoff polling loop has a max iteration count; closes a path where a buggy BIOS could loop forever.
- **PCI MSI-X vector count capped per-HC** — controllers advertising absurd vector counts (e.g. 256) get clamped to a reasonable max; closes a resource-exhaustion path.
- **DbC connect rate-limit** — DbC connection establishment rate-limited per HC; closes a "attacker plugs/unplugs DbC cable 10K times/s to OOM the connection-handling work" class.
- **xhci-sideband (audio offload) gated** — sideband ops for USB audio offload to DSP require CAP_SYS_ADMIN; in-VM containers cannot use sideband on hostUSB.

Per-doc rationale: xHCI is the cross-trust-boundary between hardware (untrusted USB devices, untrusted external xHCI silicon) and host kernel memory. BadUSB attacks (HID keystroke injection from rogue cable), Thunderclap (DMA from rogue device), DbC (kernel-debug-over-cable) all converge here. The Rust translation's typed ring + typed EP/slot state machines close the largest historical bug classes structurally (the ring-bookkeeping races that have produced multiple CVEs over the years). The IOMMU-mandatory gate is structural — refusing to drive xHCI without per-device DMA isolation is the right posture for a USB host in 2026. The DbC + LOCKDOWN integration is structural — DbC is too powerful to leave behind anything weaker than full lockdown control.

## Open Questions

- [ ] Q1: Should Rookery support xHCI without IOMMU at all? Upstream allows it (with a warning). Rookery could refuse entirely; trade-off is breaking pre-IOMMU x86 + some embedded ARM platforms. Recommendation: refuse on x86_64 (always has IOMMU); allow with explicit cmdline opt-in on embedded where IOMMU isn't always present.
- [ ] Q2: DbC enabled by default or opt-in? Upstream is Kconfig-gated (CONFIG_USB_XHCI_DBGCAP). Rookery should default OFF; opt-in via kernel cmdline + LOCKDOWN bypass.
- [ ] Q3: SoC variants — MTK, Tegra, R-Car etc. Should they be separate Tier-3s or covered under the same? They share the generic core but have substantial vendor-specific code. Recommendation: cover the generic + xhci-pci here; SoC variants get short follow-up docs if their behavior diverges materially.
- [ ] Q4: Bulk streams — usable in production today only for UAS (USB Attached SCSI) and a handful of other devices. Should Rookery implement full 64K-stream support or cap at 256 (covers UAS)? Recommendation: full support — the surface is small once the typed-ring is correct.
- [ ] Q5: USB Type-C / PD integration — role-swap, alt-mode, USB4 over USB-C. Largely happens at the PD-stack + UCSI layer, not in xHCI itself. Out of scope for this doc.
- [ ] Q6: USB sideband audio offload (`xhci-sideband.c`) — recently added for offloading USB audio DSP work to dedicated silicon. Niche; cover here or punt?

## Verification

- **Kani SAFETY**: prove that a TRB written with the wrong cycle bit cannot enqueue (typestate). Prove that a Stop-EP can only be issued on an EP in `Running` or `Halted` state. Prove the slot_id validation cannot be bypassed in any event-handling path.
- **TLA+**: model the EP state machine with concurrent URB enqueue + STOP_ENDPOINT cmd + RESET_ENDPOINT cmd. Check no path produces an EP in two states simultaneously. Check that all completion events for a stopped EP arrive before the EP transitions to `Stopped` (no late completions causing UAF).
- **Verus**: functional spec of URB→TRB lowering — for any valid URB (bulk/control/intr/isoc) with valid sg-list, produces a TRB sequence that USB-spec-conforms; the cycle bit pattern matches the current PCS at write time.
- **Kani+Verus**: invariant that every URB submitted has a tracked TRB sequence on the EP ring; every completion event maps to a tracked URB; no leaks at EP destroy.
- **Integration**: USB-IF compliance test suite (USB 3 SuperSpeed + SuperSpeedPlus); BadUSB resistance test (rogue HID device that injects 10K keystrokes/s — IOMMU + URB-rate-limit should mitigate); plug/unplug stress (10K cycles on a single port); DbC end-to-end with two machines; suspend/resume across all chip variants in test matrix.
- **Fuzz**: TRB event fuzzer (synthesized event TRBs with mutated slot_id, EP index, TRB pointer) — must validate and reject without panic. PORTSC register fuzzer (mutate port-status bits) — roothub-control must handle gracefully.
- **Penetration**: rogue USB device with DMA attack attempts — IOMMU enforcement must prevent host-memory read/write outside the device's permitted DMA window. DbC connection attempts from non-root — refused.

## Out of Scope

- USB core (`drivers/usb/core/*`) — separate Tier-3 (`drivers/usb/00-overview.md` already covered)
- EHCI/OHCI/UHCI (legacy USB host controllers) — separate Tier-3s (legacy, lower priority)
- USB device drivers (HID, storage, audio, video, network) — each its own Tier-3
- USB Type-C / PD stack (`drivers/usb/typec/*`) — separate Tier-3
- USB gadget (`drivers/usb/gadget/*`) — separate Tier-3 (device-mode)
- UCSI (`drivers/usb/typec/ucsi/*`) — separate
- xHCI SoC variants (MTK BW scheduler, Tegra FW partition) — covered briefly here; may get follow-up docs if behavior diverges materially
- xhci-sideband — punt per Q6
- DbC TTY-side userspace tooling — out of scope
