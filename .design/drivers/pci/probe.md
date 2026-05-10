# Tier-3: drivers/pci/probe.c — PCI bus enumeration (recursive scan + device creation + capability scan + ARI/MFD)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/pci/00-overview.md
upstream-paths:
  - drivers/pci/probe.c
  - drivers/pci/pci.h
  - include/linux/pci.h
-->

## Summary

PCI/PCIe bus enumeration: starting from each host bridge, recursively walk Type-1 PCI-to-PCI bridges discovering downstream devices via Configuration-Space reads of vendor/device id at every (bus, dev, func). For every responding device: read class/subclass/programming-interface, sub-device-vendor/sub-device-id, BAR sizes (write 0xFF, read back, mask), capability-list (standard caps under PCI_CAPABILITY_LIST + extended caps in 4KB ECAM region), allocate `struct pci_dev`, register with the driver model. This Tier-3 covers `drivers/pci/probe.c` (~3600 lines).

Owns: `pci_scan_root_bus`, `pci_scan_bus`, `pci_scan_child_bus`, `pci_scan_slot`, `pci_scan_single_device`, `pci_setup_device`, `pci_alloc_dev`, `pci_release_dev`, `set_pcie_port_type`, `pci_configure_extended_tags`, `pci_init_capabilities`, `pci_get_subordinate_bus`, ARI (Alternative Routing-ID Interpretation) handling, MFD (Multi-Function Device) detection, hot-plug rescan integration.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `pci_scan_root_bus(parent, bus, ops, sysdata, &resources)` | enumerate from a root bridge | `Bus::scan_root` |
| `pci_scan_root_bus_bridge(bridge)` | newer entry-point that takes pci_host_bridge directly | `HostBridge::scan` |
| `pci_scan_bus(bus, ops, sysdata)` | scan a bus at given num | `Bus::scan` |
| `pci_scan_child_bus(bus)` | recurse downstream of a bus | `Bus::scan_children` |
| `pci_scan_child_bus_extend(bus, available_buses)` | recurse with bus-range hint | `Bus::scan_children_extend` |
| `pci_scan_slot(bus, devfn)` | scan all 8 functions at one device-slot | `Bus::scan_slot` |
| `pci_scan_single_device(bus, devfn)` | scan one (bus, dev, func) | `Bus::scan_single_device` |
| `pci_alloc_dev(bus)` | allocate empty pci_dev | `PciDev::alloc` |
| `pci_setup_device(dev)` | populate dev fields from config-space | `PciDev::setup` |
| `pci_release_dev(dev)` | inverse of alloc | `PciDev::release` (Drop) |
| `pci_init_capabilities(dev)` | scan + init standard + extended caps | `PciDev::init_capabilities` |
| `pci_setup_bridge(bus)` | configure bridge windows | `Bus::setup_bridge` |
| `pci_assign_unassigned_resources()` | per-host BAR assignment (cross-ref `setup-bus.md`) | `HostBridge::assign_resources` |
| `pci_register_host_bridge(bridge)` | register a pci_host_bridge | `HostBridge::register` |
| `pci_create_root_bus(parent, bus, ops, sysdata, &resources)` | construct root bus | `Bus::create_root` |
| `pci_add_device(bus, dev)` | add device to bus, register with driver model | `Bus::add_device` |
| `pci_bus_assign_domain_nr(bus, parent)` | assign PCI domain (segment) number | `Bus::assign_domain_nr` |
| `pci_msi_setup_pci_dev(dev)` | per-device MSI/MSI-X capability init | `PciDev::msi_setup` |
| `pci_aer_init(dev)` | per-device AER cap init | `PciDev::aer_init` |
| `pci_dpc_init(dev)` | per-device DPC init | `PciDev::dpc_init` |
| `pci_iov_init(dev)` | per-device SR-IOV cap init | `PciDev::iov_init` |
| `pci_ats_init(dev)` | per-device ATS cap init | `PciDev::ats_init` |
| `pci_pri_init(dev)` | per-device PRI cap init | `PciDev::pri_init` |
| `pci_pasid_init(dev)` | per-device PASID cap init | `PciDev::pasid_init` |
| `pci_acs_init(dev)` | per-device ACS cap init | `PciDev::acs_init` |
| `pci_ptm_init(dev)` | per-device PTM cap init | `PciDev::ptm_init` |
| `pci_ari_init(dev)` | per-device ARI cap init | `PciDev::ari_init` |
| `set_pcie_port_type(dev)` | classify PCIe port type (Endpoint, Root Port, Upstream Switch Port, Downstream Switch Port, Root Complex Endpoint, Root Complex Integrated Endpoint, Root Complex Event Collector) | `PciDev::set_port_type` |

## Compatibility contract

REQ-1: Recursive scan visits every responding (bus, dev, func) combination present in upstream baseline; total device count + per-device (vendor, device, sub_vendor, sub_device, class, devfn, bus.number, bus.subordinate) matches.

REQ-2: BAR sizing per PCI Local Bus Spec § 6.2.5.1: write 0xFFFFFFFF (or 0xFFFFFFFFFFFFFFFF for 64-bit BAR), read back, mask, decode size; preserve original value. BAR type bits + prefetchable bit decoded identically.

REQ-3: Capability list walk: standard caps (offset 0x34 → linked list under 0xC0..0xFF) terminating at 0x00 next-pointer; max walk depth bounded (defense against malformed cap-list cycles). Extended caps walk: ECAM offset 0x100 → linked list with cycle detection.

REQ-4: ARI cap detection forwards device-number range from 0x00..0xFF instead of 0x00..0x1F per ARI spec; impacts which devfns scanned downstream of an ARI-capable bridge.

REQ-5: MFD bit (bit 7 of header_type) controls whether functions 1..7 are scanned in addition to function 0; bridges + endpoints with MFD bit set scan all 8.

REQ-6: PCIe port type classified per PCIe Cap version 2 register; subsequent driver binding uses `pci_pcie_type(dev)`.

REQ-7: Per-PCI-device kobject + sysfs entry created at `/sys/bus/pci/devices/<segment>:<bus>:<dev>.<func>/` with byte-identical default attribute set (vendor, device, subsystem_vendor, subsystem_device, class, revision, irq, local_cpus, local_cpulist, numa_node, resource, resource[0..5]+rom, modalias, label, device_serial, etc.).

REQ-8: Per-device modalias `pci:v<vendor>d<device>sv<sub_vendor>sd<sub_device>bc<class>sc<subclass>i<progif>` byte-identical so depmod resolves.

REQ-9: Hot-plug rescan via `/sys/bus/pci/rescan` write or per-device `rescan` write triggers re-enumeration; new devices added without disturbing existing.

REQ-10: Per-segment domain number (PCI segment from ACPI MCFG) preserved; multi-domain systems display correctly.

## Acceptance Criteria

- [ ] AC-1: `lspci -D` output matches upstream baseline byte-identical on reference HW.
- [ ] AC-2: `lspci -vvv` decodes every per-device cap (PM, MSI, MSI-X, PCIe, AER, VC, SRIOV, ATS, PRI, PASID, ACS, LTR, PTM, DPC, ROM, DSN, MFVC, ARI, RCRB).
- [ ] AC-3: BAR-size detection: NVMe drive, 10G NIC, GPU all enumerate with correct BAR sizes (verified against vendor spec).
- [ ] AC-4: ARI test: SR-IOV-capable PF expands VF address space from 0x00..0x1F to 0x00..0xFF when ARI enabled on upstream bridge.
- [ ] AC-5: MFD test: USB3 xHCI controller with multiple functions enumerates all 4-8 functions.
- [ ] AC-6: Hot-plug test: `echo 1 > /sys/bus/pci/rescan` after `echo 0 > /sys/bus/pci/devices/<bdf>/remove` reattaches the device.
- [ ] AC-7: Multi-domain test: server with > 1 PCI segment shows per-segment device list correctly.
- [ ] AC-8: Malformed cap-list defense: cap-list with cycle detected, scan terminates with WARN; doesn't infinite-loop.

## Architecture

`Bus::scan_root(parent, bus_nr, ops, sysdata, resources)`:
1. `Bus::create_root(parent, bus_nr, ops, sysdata, resources)` allocates `struct pci_bus`.
2. `Bus::scan_children` recurses.

`Bus::scan_children(bus)`:
1. For each devfn in 0..256: `Bus::scan_slot(bus, devfn)`. Skip funcs 1..7 unless function 0 reports MFD bit OR ARI cap is present on parent.
2. For each Type-1 bridge child found: assign secondary + subordinate bus number, recurse via `Bus::scan_children`.

`Bus::scan_single_device(bus, devfn)`:
1. Read config-space u32 at offset 0 (vendor:device); 0xFFFFFFFF or 0x00000000 → no device, return.
2. `PciDev::alloc(bus)` → allocate pci_dev struct.
3. `PciDev::setup(dev)`:
   - Populate vendor, device, sub_vendor, sub_device, class, header_type, irq.
   - Decode BAR list (6 BARs for endpoints, 2 for bridges) by writing 0xFF, reading back, masking, restoring.
   - Set up `dev->resource[]` array entries.
   - Set port type via `PciDev::set_port_type(dev)`.
   - `PciDev::init_capabilities(dev)` walks std + ext cap lists.
4. `Bus::add_device(bus, dev)` registers with driver model:
   - `device_initialize(&dev->dev)`, set `dev->dev.bus = &pci_bus_type`.
   - `dev_set_name(&dev->dev, "%04x:%02x:%02x.%d", domain, bus, slot, func)`.
   - `device_add(&dev->dev)` → triggers per-bus probe + uevent.
5. Return dev.

`PciDev::init_capabilities(dev)`:
1. For each known std cap id (PM, MSI, MSI-X, PCIe, VPD, SLOT_ID, AGP, AGP3, CHSWP, SUBVENDOR, BRIDGE_PORT, DEBUG_PORT, …): `pci_find_capability(dev, id)` → walk std-cap-list.
2. Init per-cap state via `pci_<cap>_init(dev)` (msi/msix, aer, dpc, pme, ptm, iov, ats, pri, pasid, acs, …).
3. For each known ext cap id (AER, VC, SN, ARI, ATS, SRIOV, MFVC, ACS, …): walk ext-cap-list.

ARI handling: if upstream bridge has ARI cap enabled, devfns scanned 0..255 (true ARI); else 0..7 within slot 0..31.

Capability list walk safety: bounded loop (max iterations = 48 std caps, 1024 ext caps) detects malformed cycles + emits WARN. Cross-ref Layer-2 `cap_iteration.tla` in parent.

PCI domain (segment) number assigned from ACPI MCFG entries (cross-ref `pci-acpi.md` future Tier-3); on systems without MCFG, default to domain 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dev_no_uaf` | UAF | `Arc<PciDev>` outlives all Bus references; release waits for refcount==0. |
| `cap_walk_bounded` | TERMINATION | std-cap walk ≤ 48 iters; ext-cap walk ≤ 1024 iters; cycle → WARN + return. |
| `bar_no_overflow` | OVERFLOW | BAR size mask + alignment arithmetic checked. |
| `devfn_no_oob` | OOB | devfn iterated 0..255 (ARI) or 0..7 (per-slot); per-slot dev=0..31 from outer loop. |

### Layer 2: TLA+

`models/pci/cap_iteration.tla` (parent-declared): proves std + ext cap iteration always terminates.
`models/pci/bar_assignment.tla` (parent-declared): proves recursive BAR-window assignment terminates + non-overlapping.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Bus::scan_single_device(bus, devfn)` post: returned `Some(PciDev)` iff vendor:device != 0xFFFF:0xFFFF in config-space | `Bus::scan_single_device` |
| `PciDev::setup` post: `dev.bus.number == bus.number`, `dev.devfn == devfn`, all `dev.resource[i]` either populated or zero (no garbage) | `PciDev::setup` |
| `PciDev::init_capabilities` post: every supported per-cap state initialized exactly once; unsupported caps remain at default | `PciDev::init_capabilities` |

### Layer 4: Verus/Creusot functional

`Bus::scan_root(parent, bus_nr)` ↔ upstream `pci_scan_root_bus` semantic equivalence: for any reference HW, the produced `pci_bus.devices` list matches upstream byte-for-byte (same devfn order, same per-device fields). Encoded as Verus model: `forall hw. rookery_scan(hw) == upstream_scan(hw)`.

## Hardening

(Inherits row-1 features from `drivers/pci/00-overview.md` § Hardening.)

probe-specific reinforcement:

- **Capability list walk bounded** — std caps ≤ 48 iters, ext caps ≤ 1024 iters, cycle detected via "visited" bitmap (defense against malformed device producing infinite cap-list loop). Layer-2 `cap_iteration.tla` is the proof.
- **BAR sizing under per-bus lock** — concurrent device probe + already-bound device's MMIO access serialized via per-bus pci_lock (defense against in-flight DMA seeing transient 0xFFFFFFFF write during sizing).
- **devfn iteration bounded by ARI presence** — without ARI, only 0..7 funcs per slot scanned; with ARI, 0..255 devfns scanned (per-spec).
- **MFD bit checked on function 0** before scanning funcs 1..7 (defense against single-function device with stale data at funcs 1..7 causing ghost-device enumeration).
- **Per-slot lock during scan** — concurrent hot-plug rescan + initial scan don't race; per-bus + per-slot locks serialize.
- **Quirks applied at PCI_FIXUP_HEADER stage** (cross-ref `quirks.md`) — pre-resource-assignment fixups for buggy devices.
- **Per-device sysfs entry creation atomic** — entry either fully present + populated OR not present at all; partial-create rolled back on failure.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- BAR-window assignment + reassignment (covered in `setup-bus.md` + `setup-res.md` future Tier-3s)
- Config-space accessors (covered in `access.md` future Tier-3)
- Per-cap init impls (covered in `msi.md`, `pcie-aer.md`, `iov.md`, `ats.md`, `pcie-dpc.md`, etc. future Tier-3s)
- ACPI MCFG parsing (covered in `pci-acpi.md` future Tier-3)
- pci_driver bind/unbind (covered in `pci-driver.md` future Tier-3)
- Quirks (covered in `quirks.md` future Tier-3)
- Hot-plug machinery (covered in `hotplug-pciehp.md` + `hotplug-acpiphp.md` future Tier-3s)
- 32-bit-only paths
- Implementation code
