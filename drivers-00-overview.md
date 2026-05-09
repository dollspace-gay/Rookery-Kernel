---
title: "Subsystem: drivers/ — device drivers"
tags: ["design-doc", "subsystem"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-2 overview for `drivers/` — the device-driver tree. Per `00-overview.md` D3, drivers are covered in a TIERED fashion: per-driver-class overviews + per-bus designs + per-driver designs only for the v0 port set (`virtio-blk`, `virtio-net`, `virtio-console`, `e1000e`, `ahci`, `ehci/xhci`, `ps2-keyboard`, `hid-input`, `vt console`). The long tail of upstream drivers continues to use the upstream C implementation via rust-for-linux interop.

This is the **largest** subsystem in the kernel by raw line count (~70% of all kernel LoC). The strategy here is exactly what makes Rookery v0 feasible: the Driver Model is in v0, a small port set proves the model works, and the existing C drivers are loaded via the well-known rust-for-linux interop path during the multi-year transition to a fully-Rust driver tree.

### Requirements

- REQ-1: Driver model (struct device, bus_type, driver, probe/remove, devres) preserves userspace-visible layout in `/sys/{bus,class,devices}/` byte-identically.
- REQ-2: udev events are wire-identical so existing udev rules carry over.
- REQ-3: PCI bus driver (cfg-space access, BAR mgmt, MSI/MSI-X, AER, SR-IOV, ASPM) preserves userspace-visible behavior. lspci/setpci work unmodified.
- REQ-4: USB bus driver (host + hub + class drivers + gadget) preserves usbfs ABI; lsusb works.
- REQ-5: virtio bus + virtio_blk + virtio_net + virtio_console preserve their virtio-spec wire formats and `/sys/bus/virtio/` layout.
- REQ-6: ACPI handling preserves `/sys/firmware/acpi/` layout + ACPI _OSI/_OSC negotiation behavior.
- REQ-7: e1000e driver preserves observable behavior (link state, ethtool output, multi-queue + RSS).
- REQ-8: AHCI driver preserves SATA disk appearance and ATA-passthrough ioctl semantics.
- REQ-9: EHCI/xHCI USB host controllers preserve transfer scheduling + bus-protocol semantics.
- REQ-10: PS/2 keyboard driver preserves input event flow.
- REQ-11: HID-input preserves the HID-descriptor parser → input-event-device chain.
- REQ-12: VT console (`/dev/tty[1-6]`) preserves `setterm`, `getty`, framebuffer-VT integration.
- REQ-13: For each driver class NOT in the per-driver port set, the class-overview Tier-3 doc declares: which drivers continue as upstream-C-via-FFI, which migrate later, and what the FFI boundary looks like.
- REQ-14: rust-for-linux's existing driver abstractions (`rust/kernel/devres.rs`, `rust/kernel/driver.rs`, `rust/kernel/device.rs`, `rust/kernel/devid.rs`, `rust/kernel/pci.rs`, `rust/kernel/usb.rs`, `rust/kernel/virtio.rs` — many already exist) are the canonical Rust API; Rookery extends.
- REQ-15: All Tier-3 (class + bus) and Tier-4 (per-driver) docs declare unsafe-block clusters, TLA+ models (driver-model device-state machine; bus-discovery hot-plug), and Kani harnesses.

### Acceptance Criteria

- [ ] AC-1: A bare-metal Rookery boot enumerates devices identically to upstream on the same hardware (`/sys/bus/{pci,usb,virtio}/devices/` listing identical). (covers REQ-1, REQ-3, REQ-4, REQ-5)
- [ ] AC-2: `udevadm monitor` event stream during a hotplug (USB plug/unplug) is byte-identical (modulo timestamps) on Rookery vs. upstream. (covers REQ-2)
- [ ] AC-3: `lspci -vv` and `lsusb -v` output is identical (modulo dynamic data like temperature). (covers REQ-3, REQ-4)
- [ ] AC-4: `dmesg | grep -E "(virtio|virtio_blk|virtio_net|virtio_console)"` reports identical probe sequence. (covers REQ-5)
- [ ] AC-5: `acpidump` output is identical; `acpitool` works unmodified. (covers REQ-6)
- [ ] AC-6: `iperf3` over an e1000e link reaches the same throughput within 5%. (covers REQ-7)
- [ ] AC-7: `hdparm -I /dev/sda` output is identical on AHCI hardware. (covers REQ-8)
- [ ] AC-8: USB selftests under `tools/usb/` pass identically. (covers REQ-4, REQ-9)
- [ ] AC-9: `evtest` output for a PS/2 keyboard event-stream is identical. (covers REQ-10)
- [ ] AC-10: A USB HID device (e.g., a keyboard or mouse) generates identical input events on Rookery vs. upstream. (covers REQ-11)
- [ ] AC-11: `setterm`, `dmesg | head` to the VT console, `getty` running on `/dev/tty1` all work unmodified. (covers REQ-12)
- [ ] AC-12: For each non-port-set driver class: a curated reference workload using upstream-C drivers via the FFI shim succeeds. (covers REQ-13)
- [ ] AC-13: A grep over Rookery for `kernel::driver::*`, `kernel::device::*`, `kernel::pci::*`, `kernel::usb::*`, `kernel::virtio::*` shows reuse of upstream rust-for-linux abstractions. (covers REQ-14)
- [ ] AC-14: `make verify` passes drivers/ Kani harnesses; `make tla` passes drivers/ models. (covers REQ-15)

### Architecture

### Layout map

```
.design/drivers/
  00-overview.md              ← this document
  base/
    00-overview.md            ← driver model + bus_type + class + struct device + probe/remove + devres + idmapped-devices
  pci/
    00-overview.md            ← PCI bus core + cfg-space + BAR + MSI/MSI-X + AER + SR-IOV + ASPM
  usb/
    00-overview.md            ← USB bus core + host hub + class dispatcher
    host.md                   ← EHCI / xHCI common abstractions
    gadget.md                 ← USB gadget framework (lower v0 priority)
    typec.md                  ← USB Type-C subsystem
  virtio/
    00-overview.md            ← virtio bus core + transport layer + virtqueue + spec compliance
  acpi/
    00-overview.md            ← ACPI core + tables + namespace + power-management hooks
  block/
    00-overview.md            ← block driver class
    virtio-blk.md             ← (v0 port-set per-driver doc)
  net/
    00-overview.md            ← net driver class
    virtio-net.md             ← (v0)
    e1000e.md                 ← (v0)
  tty/
    00-overview.md            ← TTY + serial + console framework
    virtio-console.md         ← (v0)
    vt-console.md             ← (v0)
  ata/
    00-overview.md            ← libata + ATA driver class
    ahci.md                   ← (v0)
  usb/host/
    ehci.md                   ← (v0)
    xhci.md                   ← (v0)
  input/
    00-overview.md            ← input core + event format
    keyboard/
      atkbd.md                ← PS/2 (v0)
  hid/
    00-overview.md            ← HID class
    hid-input.md              ← HID → input event chain (v0)
  scsi/
    00-overview.md            ← SCSI subsystem (FFI'd in v0)
  nvme/
    00-overview.md            ← NVMe host + target (port subset for v0)
  mmc/
    00-overview.md            ← MMC/SD (FFI'd in v0)
  md/
    00-overview.md            ← MD/RAID + dm-mapper (FFI'd in v0)
  char/
    00-overview.md            ← char driver class (mem, random — random.c cross-ref crypto, tpm, …)
  iommu/
    00-overview.md            ← IOMMU core + Intel + AMD + virtio-iommu
  dma-buf.md                  ← cross-driver buffer sharing
  firmware.md                 ← firmware-loading framework + EFI runtime services
  clocksource.md
  clk.md
  cpufreq-cpuidle.md
  regulator.md
  power-thermal-hwmon.md
  i2c-spi-gpio.md
  rtc.md
  watchdog.md
  gpu/
    00-overview.md            ← STUB — full DRM core deferred to v1+
```

### Cross-references

- `arch/x86/00-overview.md` — `arch/x86/pci.md` covers x86-specific PCI cfg-space mechanisms.
- `block/00-overview.md` — block-driver-class consumers.
- `net/00-overview.md` — net-driver-class consumers.
- `mm/00-overview.md` — `dma-mapping` (`kernel/00-overview.md` § dma-mapping.md cross-ref).
- `crypto/00-overview.md` — `drivers/char/random.c` is the central RNG.
- `00-glossary.md` — relevant terms.

### Rust module organization (informative)

Mostly already exists in upstream rust/kernel/:
- `kernel::device`, `kernel::driver`, `kernel::devres`, `kernel::devid` — exists, extend
- `kernel::pci`, `kernel::usb`, `kernel::virtio` — exists in various states, extend
- `kernel::block` (block/00-overview cross-ref)
- `kernel::net::netdev` (net/00-overview cross-ref)
- New for v0 port-set drivers: `kernel::driver::pci::e1000e`, `kernel::driver::ata::ahci`, etc.

### Locking and concurrency

Driver model: `device->mutex`, `bus_type->p->klist_*` (klist + RCU for child enumeration), per-class lock. Hot-plug events use a notifier chain.

### Error handling

Drivers return `Result<T, KernelError>` everywhere fallible. devres-managed resources auto-release on driver detach.

### Out of Scope

- drivers/staging/ — per Q1.
- Full DRM core + per-GPU drivers in Rust — per Q2 (defer to v1+).
- Per-driver Tier-4 docs outside the v0 port set.
- Architecture-specific drivers for non-x86 arches.
- Implementation code.

### upstream references in scope (top-level driver classes)

`drivers/` has 100+ subdirectories. The Tier-3 docs spawned cover the abstractions and per-class overviews; per-driver Tier-4 docs ship for the v0 port set only (and for bus core docs).

| Category | Upstream paths | Planned design doc |
|---|---|---|
| Driver model core (struct device, bus_type, class, driver, probe/remove, devm_* devres) | `drivers/base/`, `include/linux/device.h`, `include/linux/device/{class,driver,bus}.h`, `include/linux/devres.h` | `base/00-overview.md` (Tier 3) |
| Bus: PCI / PCI-Express + PCI passthrough hosts | `drivers/pci/`, `include/linux/pci.h`, `include/uapi/linux/pci_regs.h` | `pci/00-overview.md` (Tier 3 hub) |
| Bus: USB + USB Host / Gadget / Type-C | `drivers/usb/`, `include/linux/usb.h` | `usb/00-overview.md` (Tier 3 hub) spawning `usb/{host,gadget,typec}.md` |
| Bus: virtio | `drivers/virtio/`, `include/linux/virtio.h`, `include/uapi/linux/virtio_*.h` | `virtio/00-overview.md` (Tier 3) |
| Bus: ACPI | `drivers/acpi/` | `acpi/00-overview.md` (Tier 3) |
| Bus: of (Open Firmware / DT — primarily ARM but x86 has small uses) | `drivers/of/` | folded into `acpi/00-overview.md` for v0 (x86 mostly ACPI) |
| Networking drivers (CLASS) | `drivers/net/` (~500 individual drivers) | `net/00-overview.md` (Tier 3) — class hub |
| Block drivers (CLASS) | `drivers/block/` (loop, nbd, rbd, virtio_blk, brd, drbd, null_blk, sched/, zoned/, …) | `block/00-overview.md` (Tier 3) — class hub |
| SCSI subsystem | `drivers/scsi/` | `scsi/00-overview.md` (Tier 3 hub) |
| NVMe | `drivers/nvme/` (host + target) | `nvme/00-overview.md` (Tier 3) |
| ATA/AHCI/SATA | `drivers/ata/` | `ata/00-overview.md` (Tier 3) |
| MMC/SD | `drivers/mmc/` | `mmc/00-overview.md` (Tier 3) |
| MD (RAID + multi-disk) | `drivers/md/` (md, dm, dm-*, persistent-data) | `md/00-overview.md` (Tier 3) |
| Char device misc | `drivers/char/` (`mem.c`, `random.c`, `tpm/`, …) | `char/00-overview.md` (Tier 3) |
| TTY + serial + console | `drivers/tty/` (incl. vt console, n_gsm, ldisc, serial, …) | `tty/00-overview.md` (Tier 3) |
| Input (keyboard, mouse, joystick, touchscreen) | `drivers/input/` | `input/00-overview.md` (Tier 3) |
| HID (input over USB/BT/I2C) | `drivers/hid/` | `hid/00-overview.md` (Tier 3) |
| GPU + DRM | `drivers/gpu/drm/` | `gpu/00-overview.md` (Tier 3 — STUB only in v0; DRM core port deferred to v1+) |
| IOMMU | `drivers/iommu/` (intel, amd, virtio-iommu) | `iommu/00-overview.md` (Tier 3) |
| dma-buf (cross-driver buffer sharing) | `drivers/dma-buf/` | `dma-buf.md` (Tier 3) |
| Firmware loading + EFI runtime | `drivers/firmware/` | `firmware.md` (Tier 3) |
| Hyper-V drivers (guest) | `drivers/hv/` | folded into `arch/x86/00-overview.md` § hyperv-guest.md |
| Xen drivers (guest) | `drivers/xen/` | folded into `arch/x86/00-overview.md` § xen-guest.md |
| Clocksource | `drivers/clocksource/` | `clocksource.md` (Tier 3) |
| Clock subsystem (clk) | `drivers/clk/` | `clk.md` (Tier 3) |
| cpufreq + cpuidle | `drivers/cpufreq/`, `drivers/cpuidle/` | `cpufreq-cpuidle.md` (Tier 3; cross-ref `kernel/00-overview.md` § power-mgmt) |
| Regulator | `drivers/regulator/` | `regulator.md` (Tier 3) |
| Power-supply + thermal + hwmon | `drivers/power/`, `drivers/thermal/`, `drivers/hwmon/` | `power-thermal-hwmon.md` (Tier 3) |
| I2C + SPI + GPIO | `drivers/i2c/`, `drivers/spi/`, `drivers/gpio/` | `i2c-spi-gpio.md` (Tier 3) |
| LEDs | `drivers/leds/` | folded into `power-thermal-hwmon.md` |
| RTC | `drivers/rtc/` | `rtc.md` (Tier 3) |
| Watchdog | `drivers/watchdog/` | `watchdog.md` (Tier 3) |
| MTD (flash) | `drivers/mtd/` | folded into `block/00-overview.md` (cross-ref) for v0 |
| UFS (Universal Flash Storage) | `drivers/ufs/` | folded into `scsi/00-overview.md` |
| Misc | `drivers/misc/` | folded into per-class as appropriate |
| Staging | `drivers/staging/` | OUT OF SCOPE for v0 design (see Q below) |

### v0 port set (per `00-overview.md` D3): per-driver Tier-4 docs

```
.design/drivers/
  block/
    virtio-blk.md
  net/
    virtio-net.md
    e1000e.md
  tty/
    virtio-console.md
    vt-console.md
  ata/
    ahci.md
  usb/
    host/
      ehci.md
      xhci.md
  input/
    keyboard/
      atkbd.md     ← PS/2
  hid/
    hid-input.md
```

(All other drivers covered by their class doc; not separately documented.)

### compatibility contract

### Userspace-visible driver model

- **`/sys/bus/<bus>/{devices,drivers}/`** layout — preserved per per-bus docs.
- **`/sys/class/<class>/`** layout — preserved per per-class docs.
- **`/sys/devices/`** physical-topology layout — preserved.
- **udev events** (uevent format) — wire-identical so existing udev rules apply.
- **`/dev/<node>` major:minor numbers** per `Documentation/admin-guide/devices.txt` — preserved.
- **`/proc/devices`, `/proc/partitions`, `/proc/iomem`, `/proc/ioports`** — format-identical.

### Per-bus userspace-visible

- **PCI**: lspci output (parsing `/sys/bus/pci/devices/*/...`); BAR layout in `/sys/bus/pci/devices/*/resource*`; config-space ABI via `/sys/bus/pci/devices/*/config`; `setpci` works.
- **USB**: `lsusb` output (parsing `/sys/bus/usb/devices/*/`); usbfs (mounting at `/dev/bus/usb/`).
- **virtio**: `lspci -v` reports virtio devices identically; `/sys/bus/virtio/devices/*` layout.
- **ACPI**: `/sys/firmware/acpi/`, `/sys/firmware/acpi/tables/`.
- **DRI**: `/dev/dri/cardN`, `/dev/dri/renderDN`, KMS/DRM ioctls — preserved IF GPU/DRM is in v0 (it isn't — v1+).

### Per-driver-in-v0-port-set

- **virtio-blk**: appears as `/dev/vda` etc.; same ioctls as a regular block device.
- **virtio-net**: appears as `eth*` / `enp*s*` netdev; ethtool returns identical-format data.
- **virtio-console**: appears as `/dev/hvc0` / `/dev/vport0p1`.
- **e1000e**: `eth*` netdev, identical ethtool output.
- **AHCI**: SATA disks visible as `/dev/sd*`.
- **xHCI/EHCI**: USB host controllers; bus-discovery enumerates devices identically.
- **PS/2**: `event0` + `event1` typically (input event format).
- **HID-input**: HID-class-driven devices (USB / Bluetooth / I2C-HID) appear as input event devices.
- **vt console**: `/dev/tty[1-6]` virtual consoles; same VT-IOC ioctls.

### verification

### Layer 1: Kani SAFETY proofs
- DMA-buffer mapping
- IRQ-handler registration
- BAR ioremap

### Layer 2: TLA+ models
- `models/drivers/device_state.tla` — proves the driver-model device-state machine (PROBE → BIND → … → REMOVE) honors invariants under hot-plug.
- `models/drivers/bus_discovery.tla` — proves bus-rescan + driver-bind correctness under racing hot-plug.

### Layer 3: Kani harnesses
- Driver-bound device tracking (no double-bind)
- Per-driver state machines (per-driver Tier-4 docs flesh these out)

### Layer 4: opt-in
- USB-descriptor parsing (HID-descriptor especially) via Creusot.
- virtio-spec compliance proofs.

### hardening

Placeholder per `00-overview.md` D6. Notable: drivers are a top CVE source historically; the formal-verification baseline + reduced-`unsafe` Rust drivers are the project's response.

