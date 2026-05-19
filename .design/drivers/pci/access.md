# Tier-3: drivers/pci/{access,ecam}.c — PCI configuration-space accessors (CF8/CFC + ACPI MMCONFIG ECAM + per-controller ops)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/pci/00-overview.md
upstream-paths:
  - drivers/pci/access.c
  - drivers/pci/ecam.c
  - drivers/pci/pci.h
  - include/linux/pci.h
  - include/linux/pci-ecam.h
-->

## Summary

Per-device PCI configuration-space access — read/write 1/2/4 byte values at byte offset 0..255 (legacy PCI config-space) or 0..4095 (PCIe extended config-space). Two backends:
- **Legacy CF8/CFC port-IO** (x86): `outl(addr_to_cf8(bus, dev, fn, reg), CF8); inl(CFC)` for read; same form for write. Limited to first 256 bytes per device.
- **MMCONFIG / ECAM** (PCIe-spec ACPI MCFG-described): `__raw_readl(MMIO_BASE + (bus * 256 + dev * 8 + fn) * 4096 + reg)` for read; same for write. Supports full 4 KB extended config-space.

Per-host-bridge `pci_ops` vtable selects the backend; per-device `pci_read_config_*` / `_write_config_*` consumer-side calls dispatch through the parent host-bridge's ops.

This Tier-3 covers `drivers/pci/access.c` (config-space accessor wrappers + bit-aligned helpers + per-device sysfs `config` file) + `drivers/pci/ecam.c` (per-host-bridge ECAM mapping + generic-ECAM `pci_ops` impl).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pci_ops` | per-host-bridge ops vtable (`read`, `write`, `add_bus`, `remove_bus`) | `kernel::pci::PciOps` (trait) |
| `struct pci_config_window` | per-host-bridge MMIO ECAM region | `kernel::pci::ConfigWindow` |
| `pci_read_config_byte(dev, where, val)` / `_word` / `_dword` | per-device read | `PciDev::config_read_*` |
| `pci_write_config_byte(dev, where, val)` / `_word` / `_dword` | per-device write | `PciDev::config_write_*` |
| `pci_user_read_config_byte(dev, where, val)` / `_word` / `_dword` | sysfs-accessed read (with cap check) | `PciDev::user_config_read_*` |
| `pci_user_write_config_byte(...)` / `_word` / `_dword` | sysfs-accessed write | `PciDev::user_config_write_*` |
| `pci_bus_read_config_byte(bus, devfn, where, val)` / `_word` / `_dword` | per-bus accessor (used pre-pci_dev creation) | `Bus::config_read_*` |
| `pci_bus_write_config_byte(...)` / `_word` / `_dword` | per-bus write | `Bus::config_write_*` |
| `pci_generic_config_read(bus, devfn, where, size, val)` / `_write` | generic ECAM read/write | `Ecam::generic_read` / `_write` |
| `pci_generic_config_read32(...)` / `_write32` | 32-bit-aligned ECAM (some hardware requires aligned access) | `Ecam::generic_read32` / `_write32` |
| `pci_ecam_create(dev, cfgres, busr, ops)` | construct ConfigWindow from MMIO resource | `ConfigWindow::create` |
| `pci_ecam_free(cfg)` | destroy ConfigWindow | `ConfigWindow::free` |
| `pci_host_common_probe(pdev)` | generic ECAM-host-bridge probe | `EcamHostBridge::probe` |
| `pci_host_common_remove(pdev)` | inverse | `EcamHostBridge::remove` |
| `pci_ecam_map_bus(bus, devfn, where)` | translate (bus, devfn, where) → MMIO addr | `Ecam::map_bus` |
| `pci_lock_rescan_remove()` / `_unlock_rescan_remove()` | global lock for hotplug rescan-vs-remove | `Subsystem::lock_rescan_remove` |

## Compatibility contract

REQ-1: `pci_read_config_byte(dev, where, &val)` returns 0 + reads byte at config-space offset `where` from device; per-host-bridge backend dispatches transparently.

REQ-2: ECAM region per-bus-stride = 256 KB (each bus consumes 256 devices × 8 functions × 4 KB = 8 MB; standard ECAM segment is 256 buses = 256 MB).

REQ-3: Legacy CF8/CFC backend supports first 256 bytes per (bus, devfn); attempts to access offset >= 256 fail with -EINVAL.

REQ-4: ECAM backend supports full 4 KB extended config-space per (bus, devfn).

REQ-5: Per-host-bridge `pci_ops` source-compat for in-tree controller drivers (xilinx, layerscape, designware, mediatek, rockchip, hyperv, etc.); each implements `read` + `write` adapted to the host-bridge HW.

REQ-6: ACPI MCFG (Memory-mapped Configuration) table parsed at boot to populate per-segment ECAM regions; per-segment + per-bus-range MMIO base correctly mapped.

REQ-7: Some HW errata require 32-bit-aligned ECAM access only (`pci_generic_config_read32`); per-host-bridge quirk-table selects this variant.

REQ-8: Per-device sysfs `/sys/bus/pci/devices/<bdf>/config` binary file: read + write to it consume `pci_user_read_config_*` / `_write_config_*` (CAP_SYS_RAWIO + per-bit RO/RW filter).

REQ-9: Per-device `pci_lock_rescan_remove` synchronizes hotplug rescan + remove paths so config-space access from one path doesn't observe device mid-removal.

REQ-10: Bus-scan path (cross-ref `drivers/pci/probe.md`) uses `pci_bus_read_config_*` for pre-pci_dev-creation reads (vendor:device check at scan time).

## Acceptance Criteria

- [ ] AC-1: `lspci -xxxx` (full 4 KB config-space dump) shows ECAM-readable extended config-space on PCIe-capable system.
- [ ] AC-2: `setpci -s 00:1f.3 0x40.b=0xff` writes byte at offset 0x40 of device 00:1f.3; readback confirms.
- [ ] AC-3: ECAM access stress: 100k random config reads from 32 concurrent threads → KASAN clean, no torn read.
- [ ] AC-4: Per-host-bridge quirk test: PCH with 32-bit-aligned-only ECAM uses `pci_generic_config_read32`; byte/word reads correctly synthesized from 32-bit-aligned read.
- [ ] AC-5: Hotplug rescan-vs-remove race test: concurrent `echo 1 > /sys/bus/pci/rescan` + `echo 1 > /sys/bus/pci/devices/<bdf>/remove` → no UAF, no missed-device.
- [ ] AC-6: kselftest pci config-space subset passes.

## Architecture

`PciOps` lives in `kernel::pci::PciOps`:

```
trait PciOps: Send + Sync {
  fn read(&self, bus: &Bus, devfn: u32, where: u32, size: u32, val: &mut u32) -> Result<(), Errno>;
  fn write(&self, bus: &Bus, devfn: u32, where: u32, size: u32, val: u32) -> Result<(), Errno>;
  fn add_bus(&self, bus: &Bus) -> Result<(), Errno> { Ok(()) }
  fn remove_bus(&self, bus: &Bus) {}
}
```

`ConfigWindow` (ECAM region) lives in `kernel::pci::ConfigWindow`:

```
struct ConfigWindow {
  refcount: Refcount,
  parent: Arc<Device>,
  resource: Resource,                  // MMIO base+size
  bus_range: Range<u8>,
  ops: &'static dyn PciOps,
  win: NonNull<u8>,                     // memremap'd MMIO virtual addr
  priv: KBox<dyn Any>,
}
```

Legacy CF8/CFC backend (x86):
```
fn pci_x86_legacy_read(bus, devfn, where, size, val):
  raw_spin_lock(&pci_config_lock)
  outl((1<<31) | (bus<<16) | (devfn<<8) | (where & 0xfc), CONFIG_ADDRESS=0xCF8)
  match size {
    1 => *val = inb(CONFIG_DATA + (where & 3))
    2 => *val = inw(CONFIG_DATA + (where & 2))
    4 => *val = inl(CONFIG_DATA)
  }
  raw_spin_unlock(&pci_config_lock)
```

ECAM backend `Ecam::generic_read`:
```
fn pci_generic_config_read(bus, devfn, where, size, val):
  if where + size > cfg.bus_range_size_per_dev { return -EINVAL }
  let addr = ecam_map_bus(cfg, bus.number, devfn, where);
  match size {
    1 => *val = readb(addr)
    2 => *val = readw(addr)
    4 => *val = readl(addr)
  }
  Ok(())
```

`pci_ecam_map_bus(bus, devfn, where)`:
```
fn ecam_map_bus(cfg, bus_num, devfn, where):
  if !(cfg.bus_range.contains(&bus_num)) { return ptr::null() }
  let bus_off = (bus_num - cfg.bus_range.start) * (1 << 20);  // 1 MB per bus
  let dev_off = devfn << 12;                                   // 4 KB per devfn
  cfg.win.as_ptr() + bus_off + dev_off + where
```

`pci_host_common_probe(pdev)` for generic ECAM platform-driver:
1. Read MMIO `cfgres` resource from DT/ACPI.
2. Parse bus-range from DT/ACPI.
3. `ConfigWindow::create(dev, cfgres, busr, &pci_generic_ecam_ops)` → memremap MMIO + populate cfg.
4. `pci_host_probe(host_bridge)` → triggers full enumeration via `Bus::scan_root` (cross-ref `drivers/pci/probe.md`).

User-mediated config-space access (`pci_user_*_config_*`):
1. Validate `where + size <= 4096` (full ECAM).
2. CAP_SYS_RAWIO check.
3. Per-bit access-control filter (cross-ref `drivers/pci/00-overview.md` § Hardening).
4. Dispatch via parent host-bridge's `ops`.

Per-`/sys/bus/pci/devices/<bdf>/config` sysfs binary file:
- read(2) → `pci_read_config_dword(dev, pos, &val)` for each 4-byte chunk.
- write(2) → CAP_SYS_RAWIO check + `pci_user_write_config_*` (with per-bit filter).

Hotplug synchronization: `pci_lock_rescan_remove()` is a global mutex held during `pci_scan_root_bus_bridge` + `pci_stop_and_remove_bus_device`; ensures no concurrent rescan and remove operating on overlapping device sets.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ecam_no_oob` | OOB | `ecam_map_bus` validates bus_num, devfn, where against ECAM region; returns null on out-of-range. |
| `legacy_no_oob` | OOB | Legacy CF8/CFC backend validates `where < 256`; returns -EINVAL otherwise. |
| `cfg_lock_no_deadlock` | DEADLOCK | per-host-bridge config lock acquired in canonical order (per-bus, never cross-bus); deadlock-free. |
| `mmio_addr_aligned` | ALIGNMENT | per-size MMIO accessor invoked at correctly-aligned addr (size-1) | per-spec | `Ecam::generic_read` |

### Layer 2: TLA+

`models/pci/cap_iteration.tla` (parent-declared): proves std + ext capability iteration always terminates given any 4 KB config-space content.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `pci_read_config_*` post: returned val matches device's actual config-space byte at `where` (under no-write-since-last-read invariant) | `PciDev::config_read_*` |
| `pci_write_config_*` post: device's config-space byte at `where` updated to `val` (modulo per-bit RO/passthrough/virtualization) | `PciDev::config_write_*` |
| `pci_user_*_config_*` invariant: per-bit filter applied + CAP_SYS_RAWIO checked + per-LSM hook invoked | `PciDev::user_config_*` |

### Layer 4: Verus/Creusot functional

`pci_read_config_dword(dev, where, &val)` ↔ `pci_write_config_dword(dev, where, val)` round-trip equivalence: write at offset `where` with value `val` followed by read returns `val` (modulo per-bit RO bits which preserve the original value).

## Hardening

(Inherits row-1 features from `drivers/pci/00-overview.md` § Hardening.)

access-specific reinforcement:

- **Config-space write CAP_SYS_RAWIO** required for `/sys/bus/pci/devices/<bdf>/config` writes (defense against userspace re-flashing PCI ROM, programming MSI vectors, etc.).
- **Per-bit RO/RW/passthrough/virtualization filter** applied to user writes (defense against userspace bypassing per-cap RO bits).
- **CF8/CFC legacy backend port-IO ratelimit** — port-IO from non-init userns gated by CAP_SYS_RAWIO + LSM (defense against generic CAP_SYS_RAWIO not implying CF8/CFC access in container scenarios).
- **ECAM mapping uses memremap with explicit cache-attribute** — UC for legacy x86 chipsets, WC where supported by spec; defense against cache-attribute-mismatch causing torn reads.
- **Per-host-bridge config_lock** — ensures atomic 4-byte access to single PCI config register across CPU contention.
- **Capability list walk bounded** at 48 std + 1024 ext iterations per `Bus::scan_single_device` (cross-ref `drivers/pci/probe.md`); defense against malformed cap-list cycle DoS.
- **Per-device `pci_lock_rescan_remove`** — hotplug rescan + remove synchronized via global mutex; defense against concurrent operations causing UAF.
- **Generic-ECAM region MMIO bounds-checked** — ECAM region's MMIO range validated against ACPI MCFG-declared range before memremap.
- **Per-host-bridge ops vtable read-only-after-init** — defense against runtime modification.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — /sys/bus/pci/devices/<dev>/config read/write buffer and pci_vpd slab whitelisted; defense against per-oversized-config-read leaking adjacent slab.
- **PAX_KERNEXEC** — pci_bus_read_config_{byte,word,dword}, pci_user_read_config_*, raw_pci_ops dispatch run W^X.
- **PAX_RANDKSTACK** — per-/sys/bus/pci/.../config-read syscall entry randomizes kernel-stack offset; defense against per-config-space-side-channel.
- **PAX_REFCOUNT** — struct pci_dev, struct pci_host_bridge, struct pci_bus refs saturating refcount_t; defense against per-hotplug-storm refcount overflow.
- **PAX_MEMORY_SANITIZE** — pci_vpd cache, pci_saved_state, pci_cap buffers poison-on-free; defense against per-prior-device-config leak across hot-unplug/replug.
- **PAX_UDEREF** — pci_read_config / pci_write_config sysfs path enforces split user/kernel on copy_to_user of raw config space.
- **PAX_RAP/kCFI** — pci_ops->read / pci_ops->write and pci_host_bridge ops vtables CFI-protected; defense against per-forged-pci_ops via crafted host-bridge driver-load race.
- **GRKERNSEC_HIDESYM** — raw_pci_ops, pci_root_buses, per-host-bridge ops addresses hidden from /proc/kallsyms.
- **GRKERNSEC_DMESG** — "pci %s: BAR %d", "config-space oops" prints restricted; defense against per-topology-disclosure.
- **Config-space CAP_SYS_RAWIO** — read/write of /sys/bus/pci/devices/<dev>/config (and PCIIOC_* on /proc/bus/pci/<bus>/<dev>) gated on CAP_SYS_RAWIO; defense against per-unprivileged scraping of subsystem-id / device-id / Vendor-Specific Caps for fingerprinting and per-unprivileged write of BAR / MSI vectors / extended caps.
- **Config-space write length bounded** — only PCI_CFG_SPACE_EXP_SIZE (4096) bytes addressable; defense against per-oversized pwrite walking off PCIe ECAM region.
- **Per-bus pci_ops generation count** — config-space access serialized; defense against per-bus-rescan-race delivering stale ops to a concurrent reader.
- **Generic-ECAM MMIO bounds-checked vs MCFG** — defense against per-rogue host-bridge driver providing an ECAM window outside the firmware-declared range.
- **Per-pci_user_read_config retry on hardware-error** — bounded to PCI_USER_READ_RETRIES; defense against per-broken-device hanging the config-space reader.
- Rationale: PCI config space is the device-discovery and -control plane; grsec posture is dominated by CAP_SYS_RAWIO at the sysfs/procfs boundary, usercopy-whitelisting and length bounds on the config-space slab, CFI on pci_ops dispatch, and sanitize-on-free of saved-state across hotplug.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- ACPI MCFG table parsing details (covered in `drivers/pci/pci-acpi.md` future Tier-3)
- Per-host-bridge controller drivers (covered in `drivers/pci/controllers-*.md` future Tier-3s — most ARM SoC, compile-gated off for v0)
- Per-cap content interpretation (covered in `drivers/pci/probe.md` parent)
- Per-bit virt/RO/passthrough table for vfio-pci (covered in `drivers/vfio/pci-config.md` future Tier-3)
- 32-bit-only paths
- Implementation code
