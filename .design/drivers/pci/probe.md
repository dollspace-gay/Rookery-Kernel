# Tier-3: drivers/pci/probe.c — PCI/PCIe bus enumeration

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/pci/00-overview.md
upstream-paths:
  - drivers/pci/probe.c (~3585 lines)
  - drivers/pci/pci.h
  - include/linux/pci.h
  - include/uapi/linux/pci_regs.h
-->

## Summary

PCI/PCIe bus enumeration walks the topology starting from each host bridge, recursing through Type-1 PCI-to-PCI bridges and reading Configuration Space at every (bus, device, function) triple. For each responding device the enumerator allocates a `struct pci_dev`, populates vendor/device/class/header-type/subsystem from config space, sizes every Base Address Register (BAR) by the write-0xFF / read-back / mask algorithm, classifies PCIe port type from the PCIe Capability, opens MSI/MSI-X capability stubs (disabled-on-probe), discovers ARI / SR-IOV / ACS / AER / PASID / PRI / ATS / PTM / DPC / TPH / IDE / Rebar / DOE capabilities, and registers the device with the Linux driver model so a driver can bind. For each bridge discovered, the enumerator assigns primary / secondary / subordinate bus numbers, programs the PCI_PRIMARY_BUS register, and recurses to enumerate downstream buses. Critical for: device-discovery determinism, driver-binding, hot-plug rescan, ARI / SR-IOV downstream-VF discovery, link-speed/-width reporting, MSI / MSI-X capability availability for IRQ allocation.

This Tier-3 covers `drivers/pci/probe.c` (~3585 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pci_dev` | per-device handle | `PciDev` |
| `struct pci_bus` | per-bus handle | `PciBus` |
| `struct pci_host_bridge` | per-root-bridge | `PciHostBridge` |
| `pci_scan_root_bus(parent, bus, ops, sysdata, &resources)` | per-root-scan | `Probe::scan_root_bus` |
| `pci_scan_root_bus_bridge(bridge)` | per-host-bridge-scan | `Probe::scan_root_bus_bridge` |
| `pci_scan_bus(bus, ops, sysdata)` | per-bus-scan (legacy) | `Probe::scan_bus` |
| `pci_scan_child_bus(bus)` | per-recurse-down | `Probe::scan_child_bus` |
| `pci_scan_child_bus_extend(bus, avail)` | per-recurse-with-hint | `Probe::scan_child_bus_extend` |
| `pci_scan_slot(bus, devfn)` | per-slot (8 funcs) | `Probe::scan_slot` |
| `pci_scan_single_device(bus, devfn)` | per-(bus,devfn) | `Probe::scan_single_device` |
| `pci_scan_device(bus, devfn)` | per-config-read + alloc | `Probe::scan_device` |
| `pci_alloc_dev(bus)` | per-pci_dev alloc | `PciDev::alloc` |
| `pci_release_dev(dev)` | per-pci_dev release | `PciDev::release` |
| `pci_setup_device(dev)` | per-cfg-populate | `PciDev::setup` |
| `pci_device_add(dev, bus)` | per-device-model add | `Probe::device_add` |
| `pci_scan_bridge(bus, dev, max, pass)` | per-bridge-scan | `Probe::scan_bridge` |
| `pci_scan_bridge_extend(bus, dev, max, avail, pass)` | per-bridge-with-hint | `Probe::scan_bridge_extend` |
| `pci_add_new_bus(parent, dev, busnr)` | per-new-bus-create | `Probe::add_new_bus` |
| `pci_create_root_bus(parent, bus, ops, sysdata, &resources)` | per-root-create | `Probe::create_root_bus` |
| `pci_register_host_bridge(bridge)` | per-bridge-register | `PciHostBridge::register` |
| `pci_alloc_host_bridge(priv_sz)` | per-host-bridge-alloc | `PciHostBridge::alloc` |
| `pci_free_host_bridge(bridge)` | per-host-bridge-free | `PciHostBridge::free` |
| `pci_host_probe(bridge)` | per-host-bridge-probe + assign + add | `Probe::host_probe` |
| `pci_bus_read_dev_vendor_id(bus, devfn, &l, timeout)` | per-vendor-id-read + RRS-wait | `PciBus::read_dev_vendor_id` |
| `pci_bus_generic_read_dev_vendor_id(...)` | per-vendor-id-read inner | `PciBus::read_dev_vendor_id_generic` |
| `__pci_read_base(dev, type, res, pos, sizes)` | per-BAR-decode | `PciDev::read_base` |
| `__pci_size_stdbars(dev, count, pos, stdbars)` | per-BAR-size-probe (write 0xFF, read) | `PciDev::size_stdbars` |
| `pci_read_bridge_bases(child)` | per-bridge-window-decode | `PciBus::read_bridge_bases` |
| `pci_cfg_space_size(dev)` | per-cfg-space-size detect (256 vs 4096) | `PciDev::cfg_space_size` |
| `set_pcie_port_type(pdev)` | per-PCIe-port-type | `PciDev::set_pcie_port_type` |
| `set_pcie_hotplug_bridge(pdev)` | per-hotplug-bridge-detect | `PciDev::set_pcie_hotplug` |
| `pci_init_capabilities(dev)` | per-cap-init (std + ext) | `PciDev::init_capabilities` |
| `pci_msi_init(dev)` | per-MSI-cap discover + disable | `PciDev::msi_init` |
| `pci_msix_init(dev)` | per-MSI-X-cap discover + disable | `PciDev::msix_init` |
| `pci_configure_ari(dev)` | per-ARI-enable on bridge | `PciDev::configure_ari` |
| `pci_iov_init(dev)` | per-SR-IOV-cap init | `PciDev::iov_init` |
| `pci_acs_init(dev)` | per-ACS-cap init | `PciDev::acs_init` |
| `pci_aer_init(dev)` | per-AER-cap init | `PciDev::aer_init` |
| `pci_dpc_init(dev)` | per-DPC-cap init | `PciDev::dpc_init` |
| `pci_ats_init(dev)` | per-ATS-cap init | `PciDev::ats_init` |
| `pci_pri_init(dev)` | per-PRI-cap init | `PciDev::pri_init` |
| `pci_pasid_init(dev)` | per-PASID-cap init | `PciDev::pasid_init` |
| `pci_ptm_init(dev)` | per-PTM-cap init | `PciDev::ptm_init` |
| `pcie_get_supported_speeds(dev)` | per-supported-link-speeds | `PciDev::supported_speeds` |
| `pcie_update_link_speed(bus, reason, linksta, linksta2)` | per-bus-link-speed | `PciBus::update_link_speed` |
| `pci_configure_extended_tags(dev, ign)` | per-extended-tags | `PciDev::configure_extended_tags` |
| `pci_configure_mps(dev)` | per-Max-Payload-Size | `PciDev::configure_mps` |
| `pcie_bus_configure_settings(bus)` | per-bus-MPS-settings | `PciBus::configure_settings` |
| `pci_set_msi_domain(dev)` | per-MSI-irq-domain attach | `PciDev::set_msi_domain` |
| `pci_hp_add_bridge(dev)` | per-hotplug-bridge-rescan | `Probe::hp_add_bridge` |
| `pci_rescan_bus(bus)` | per-bus-rescan trigger | `PciBus::rescan` |
| `pci_lock_rescan_remove()` / `_unlock_` | per-rescan-remove serialize | `Probe::rescan_remove_lock` |
| `pci_bus_insert_busn_res(b, bus, bus_max)` | per-bus-num-resource insert | `PciBus::insert_busn_res` |
| `pci_bus_update_busn_res_end(b, bus_max)` | per-bus-num-resource resize | `PciBus::update_busn_res_end` |
| `pci_bus_release_busn_res(b)` | per-bus-num-resource release | `PciBus::release_busn_res` |
| `pci_bus_assign_domain_nr(bus, parent)` | per-PCI-segment-number | `PciBus::assign_domain_nr` |

## Compatibility contract

REQ-1: struct pci_dev (per-device fields populated by probe):
- bus: per-parent pci_bus pointer.
- devfn: per-(slot, func) packed (slot << 3 | func).
- vendor / device: per-config-space u16 at 0x00/0x02.
- subsystem_vendor / subsystem_device: per-config-space u16 at 0x2C/0x2E (normal hdr).
- class: per-config-space u24 = (revision << 24) >> 8 from PCI_REVISION_ID/PCI_CLASS_REVISION.
- revision: per-config-space u8 at 0x08.
- hdr_type: per-config-space u8 at 0x0E, masked PCI_HEADER_TYPE_MASK.
- multifunction: per-MFD-bit (bit 7 of hdr_type byte).
- cfg_size: per-cfg-space-size (256 or 4096).
- pcie_cap: per-offset of PCIe Capability (PCI_CAP_ID_EXP) or 0.
- msi_cap / msix_cap: per-offset of MSI / MSI-X Capability or 0.
- pcie_flags_reg: per-PCIe-Cap u16 flags at PCI_EXP_FLAGS.
- pcie_mpss: per-Max-Payload-Size-Supported (PCI_EXP_DEVCAP[2:0]).
- supported_speeds: per-PCIe-Cap supported link-speeds bitmap.
- link_active_reporting: per-LNKCAP-DLLLARC bit.
- aspm_l0s_support / aspm_l1_support: per-LNKCAP ASPM bits.
- is_hotplug_bridge / is_pciehp: per-SLTCAP-HPC bit.
- is_thunderbolt: per-VSEC PCI_VSEC_ID_INTEL_TBT.
- is_cxl: per-DVSEC CXL FlexBus.
- untrusted: per-external-facing or arch_pci_dev_is_removable.
- non_compliant_bars / mmio_always_on / broken_intx_masking: per-quirk flags.
- error_state: pci_channel_io_normal at probe.
- dma_mask: per-default 0xffffffff (32-bit).
- msi_addr_mask: per-default DMA_BIT_MASK(64) (32-bit downgrade on PCI_MSI_FLAGS_64BIT cleared).
- resource[0..PCI_NUM_RESOURCES-1]: per-BAR + ROM + bridge windows.
- bus_list: per-bus->devices list entry.
- pcie_cap_lock: per-PCIe-Cap RMW lock.
- msi_lock: per-MSI-mask RMW raw_spinlock_t.

REQ-2: Probe::scan_single_device(bus, devfn):
- /* Already added? */
- existing = pci_get_slot(bus, devfn). If Some: pci_dev_put; return existing.
- /* Read vendor/device with RRS-wait */
- if !PciBus::read_dev_vendor_id(bus, devfn, &l, 60_000_ms): return None.
- /* Sanity: 0x0000ffff / 0xffff0000 / 0x00000000 / PCI_POSSIBLE_ERROR ⟹ empty */
- dev = PciDev::alloc(bus).
- dev.devfn = devfn; dev.vendor = l & 0xffff; dev.device = (l >> 16) & 0xffff.
- if PciDev::setup(dev) != 0: drop dev; return None.
- /* Add to driver model */
- Probe::device_add(dev, bus).
- return Some(dev).

REQ-3: PciBus::read_dev_vendor_id (RRS-aware):
- pci_bus_read_config_dword(bus, devfn, PCI_VENDOR_ID, &l).
- if PCI_POSSIBLE_ERROR(l) ∨ l ∈ {0x00000000, 0x0000ffff, 0xffff0000}: return false.
- if pci_bus_rrs_vendor_id(l):
  - /* Exponential-backoff retry up to timeout (default 60s) */
  - delay = 1.
  - while pci_bus_rrs_vendor_id(l):
    - if delay > timeout: WARN("not ready after Nms; giving up"); return false.
    - msleep(delay). delay *= 2.
    - re-read PCI_VENDOR_ID.
- return true.

REQ-4: PciDev::setup(dev):
- hdr_type_byte = config-read PCI_HEADER_TYPE (u8).
- dev.hdr_type = hdr_type_byte & PCI_HEADER_TYPE_MASK.
- dev.multifunction = (hdr_type_byte & PCI_HEADER_TYPE_MFD) != 0.
- dev.dev.bus = &pci_bus_type; dev.dev.parent = bus.bridge.
- dev.error_state = pci_channel_io_normal.
- PciDev::set_pcie_port_type(dev).
- pci_set_of_node(dev); pci_set_acpi_fwnode(dev); pci_dev_assign_slot(dev).
- dev.dma_mask = 0xffffffff; dev.msi_addr_mask = DMA_BIT_MASK(64).
- dev_set_name("%04x:%02x:%02x.%d", domain, bus.number, SLOT(devfn), FUNC(devfn)).
- class = config-read PCI_CLASS_REVISION (u32).
- dev.revision = class & 0xff; dev.class = class >> 8.
- dev.cfg_size = PciDev::cfg_space_size(dev) ∈ {256, 4096}.
- set_pcie_thunderbolt(dev); set_pcie_cxl(dev); set_pcie_untrusted(dev).
- if pci_is_pcie(dev): dev.supported_speeds = pcie_get_supported_speeds(dev).
- dev.current_state = PCI_UNKNOWN.
- pci_fixup_device(pci_fixup_early, dev); pci_set_removable(dev).
- /* Header-type-dispatch */
- match dev.hdr_type:
  - PCI_HEADER_TYPE_NORMAL:
    - if class == PCI_CLASS_BRIDGE_PCI: bad-class WARN; class = PCI_CLASS_NOT_DEFINED << 8.
    - pci_read_irq(dev).
    - pci_read_bases(dev, PCI_STD_NUM_BARS=6, PCI_ROM_ADDRESS).
    - pci_subsystem_ids(dev, &dev.subsystem_vendor, &dev.subsystem_device).
    - /* Legacy IDE quirk for class STORAGE_IDE */
  - PCI_HEADER_TYPE_BRIDGE:
    - pci_read_irq(dev).
    - dev.transparent = ((dev.class & 0xff) == 1).
    - pci_read_bases(dev, 2, PCI_ROM_ADDRESS1).
    - pci_read_bridge_windows(dev).
    - PciDev::set_pcie_hotplug(dev).
    - pos = pci_find_capability(dev, PCI_CAP_ID_SSVID); if pos: load SSVID/SSID.
  - PCI_HEADER_TYPE_CARDBUS:
    - if class != PCI_CLASS_BRIDGE_CARDBUS: bad-class.
    - pci_read_irq(dev); pci_read_bases(dev, 1, 0); SSVID/SSID at CardBus offsets.
  - default: ERR "unknown header type %02x, ignoring"; return -EIO.
- return 0.

REQ-5: PciDev::set_pcie_port_type(pdev):
- pos = pci_find_capability(pdev, PCI_CAP_ID_EXP); if !pos: return.
- pdev.pcie_cap = pos.
- pdev.pcie_flags_reg = config-read u16 at pos+PCI_EXP_FLAGS.
- type = pci_pcie_type(pdev) = FIELD_GET(PCI_EXP_FLAGS_TYPE, pdev.pcie_flags_reg).
- type ∈ {ENDPOINT, LEG_END, ROOT_PORT, UPSTREAM, DOWNSTREAM, PCI_BRIDGE, PCIE_BRIDGE, RC_END, RC_EC}.
- if type == ROOT_PORT: pci_enable_rrs_sv(pdev).
- pdev.devcap = config-read u32 at pos+PCI_EXP_DEVCAP.
- pdev.pcie_mpss = FIELD_GET(PCI_EXP_DEVCAP_PAYLOAD, pdev.devcap).
- linkcap = config-read u32 at pos+PCI_EXP_LNKCAP.
- pdev.link_active_reporting = (linkcap & PCI_EXP_LNKCAP_DLLLARC) != 0.
- if PCIEASPM: pdev.aspm_l0s_support / l1_support from LNKCAP_ASPM bits.
- /* Correct misreported upstream/downstream types */
- parent = pci_upstream_bridge(pdev).
- if type == DOWNSTREAM ∧ pcie_downstream_port(parent): pdev.pcie_flags_reg ⟵ UPSTREAM.
- if type == UPSTREAM ∧ pci_pcie_type(parent) == UPSTREAM: pdev.pcie_flags_reg ⟵ DOWNSTREAM.

REQ-6: PciDev::read_bases (BAR sizing) — `__pci_size_stdbars` + `__pci_read_base`:
- /* Disable IO/MEM decode if !mmio_always_on */
- orig_cmd = config-read u16 PCI_COMMAND.
- if orig_cmd & PCI_COMMAND_DECODE_ENABLE: config-write PCI_COMMAND (orig_cmd & ~PCI_COMMAND_DECODE_ENABLE).
- /* For each BAR pos: write 0xFFFFFFFF; read back; restore */
- __pci_size_stdbars(dev, howmany, PCI_BASE_ADDRESS_0, stdbars[]).
- if rom: __pci_size_rom(dev, rom_pos, &rombar).
- /* Restore decode */
- if orig_cmd & PCI_COMMAND_DECODE_ENABLE: config-write PCI_COMMAND orig_cmd.
- /* Decode each BAR via __pci_read_base */
- for pos in 0..howmany:
  - res = &dev.resource[pos]; reg = PCI_BASE_ADDRESS_0 + (pos << 2).
  - pos += __pci_read_base(dev, pci_bar_unknown, res, reg, &stdbars[pos]).
- /* ROM */
- if rom: res = &dev.resource[PCI_ROM_RESOURCE]; res.flags = MEM | PREFETCH | READONLY | SIZEALIGN.

REQ-7: PciDev::read_base (single BAR decode):
- /* Read low half */
- l = config-read u32 at pos. sz = sizes[0].
- if PCI_POSSIBLE_ERROR(sz): sz = 0. if PCI_POSSIBLE_ERROR(l): l = 0.
- /* IO vs MEM decode */
- if (l & PCI_BASE_ADDRESS_SPACE) == PCI_BASE_ADDRESS_SPACE_IO:
  - res.flags |= IORESOURCE_IO.
  - l64 = l & PCI_BASE_ADDRESS_IO_MASK; sz64 = sz & PCI_BASE_ADDRESS_IO_MASK; mask64 = IO_SPACE_LIMIT.
- else:
  - res.flags |= IORESOURCE_MEM.
  - if (l & PCI_BASE_ADDRESS_MEM_TYPE_MASK) == MEM_TYPE_64: res.flags |= IORESOURCE_MEM_64.
  - if (l & PCI_BASE_ADDRESS_MEM_PREFETCH): res.flags |= IORESOURCE_PREFETCH.
- res.flags |= IORESOURCE_SIZEALIGN.
- /* Read upper half for 64-bit BAR */
- if res.flags & IORESOURCE_MEM_64:
  - l_hi = config-read u32 at pos+4; sz_hi = sizes[1].
  - l64 |= ((u64) l_hi << 32); sz64 |= ((u64) sz_hi << 32); mask64 |= ((u64) ~0 << 32).
- /* sz64 == 0 ⟹ BAR not implemented */
- if !sz64: res.flags = 0; return 0.
- sz64 = pci_size(l64, sz64, mask64) /* size = ~(sz64 & mask64) + 1 (bus-region) */.
- /* 64-bit BAR > 4 GiB on 32-bit kernels ⟹ UNSET | DISABLED */
- /* Convert bus-region to CPU resource; inversion-check (resource→bus must round-trip) */
- region = { start: l64, end: l64 + sz64 - 1 }.
- pcibios_bus_to_resource(bus, res, &region); pcibios_resource_to_bus(bus, &inv, res).
- if inv.start != region.start: res.flags |= IORESOURCE_UNSET; res.start = 0; res.end = region.end - region.start; pci_info("initial BAR value invalid").
- return (res.flags & IORESOURCE_MEM_64) ? 1 : 0.

REQ-8: Probe::scan_slot(bus, devfn):
- if only_one_child(bus) ∧ devfn > 0: return 0 /* PCIe Downstream Port → only dev0 */.
- fn = 0; nr = 0.
- loop:
  - dev = Probe::scan_single_device(bus, devfn + fn).
  - if Some(dev): if !pci_dev_is_added(dev): nr += 1. if fn > 0: dev.multifunction = 1.
  - else if fn == 0: if !hypervisor_isolated_pci_functions(): break /* func0 required */.
  - fn = next_fn(bus, dev, fn). if fn < 0: break.
- /* ASPM link-state init if any device found */
- if bus.self ∧ nr > 0: pcie_aspm_init_link_state(bus.self).
- return nr.

REQ-9: next_fn / next_ari_fn (devfn iteration policy):
- if pci_ari_enabled(bus): return next_ari_fn(bus, dev, fn).
  - pos = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_ARI). if !pos: return -ENODEV.
  - cap = config-read u16 at pos+PCI_ARI_CAP. next_fn = PCI_ARI_CAP_NFN(cap).
  - if next_fn <= fn: return -ENODEV /* malformed-list defense */.
  - return next_fn.
- else /* legacy */: if fn >= 7: return -ENODEV.
- else: if dev ∧ !dev.multifunction: return -ENODEV /* func 0 only */.
- else: return fn + 1.

REQ-10: only_one_child(bus):
- /* PCIe Downstream Ports ⟹ only Device 0 on the downstream link */
- if pci_has_flag(PCI_SCAN_ALL_PCIE_DEVS): return 0.
- if bus.self ∧ pci_is_pcie(bus.self) ∧ pcie_downstream_port(bus.self): return 1.
- return 0.

REQ-11: Probe::scan_child_bus_extend(bus, available_buses):
- /* Scan all PCI_MAX_NR_DEVS=32 slots */
- for devnr in 0..PCI_MAX_NR_DEVS: Probe::scan_slot(bus, PCI_DEVFN(devnr, 0)).
- used_buses = pci_iov_bus_range(bus); max += used_buses.
- /* pcibios_fixup_bus once per bus */
- if !bus.is_added: pcibios_fixup_bus(bus); bus.is_added = 1.
- /* Count bridges (hotplug vs normal) */
- for_each_pci_bridge(dev, bus): if dev.is_hotplug_bridge: hotplug_bridges++. else: normal_bridges++.
- /* Pass 0: scan already-configured bridges */
- for_each_pci_bridge(dev, bus): max = Probe::scan_bridge_extend(bus, dev, max, 0, 0).
- /* Pass 1: scan bridges needing reconfiguration */
- for_each_pci_bridge(dev, bus): /* distribute available_buses */; max = Probe::scan_bridge_extend(bus, dev, cmax, buses, 1).
- /* If self is hotplug bridge: reserve pci_hotplug_bus_size buses */
- return max.

REQ-12: Probe::scan_bridge_extend(bus, dev, max, available_buses, pass):
- pm_runtime_get_sync(dev).
- buses = config-read u32 PCI_PRIMARY_BUS.
- primary, secondary, subordinate ← FIELD_GET on buses.
- /* If !primary ∧ secondary ∧ subordinate: hardwired-to-0, override primary = bus.number */
- /* Validate: pass-0 + (primary != bus.number ∨ secondary <= bus.number ∨ secondary > subordinate) ⟹ broken */
- bctl = config-read u16 PCI_BRIDGE_CONTROL.
- /* Disable Master-Abort during probe to avoid spurious AER */
- config-write PCI_BRIDGE_CONTROL (bctl & ~PCI_BRIDGE_CTL_MASTER_ABORT).
- if pci_is_cardbus_bridge(dev): pci_cardbus_scan_bridge_extend(...); goto out.
- if (secondary ∨ subordinate) ∧ !pcibios_assign_all_busses() ∧ !broken:
  - /* Use existing config; pass-0 only */
  - if pass: goto out.
  - child = pci_find_bus(domain, secondary). if !child: child = Probe::add_new_bus(bus, dev, secondary); pci_bus_insert_busn_res(child, secondary, subordinate); child.bridge_ctl = bctl.
  - cmax = Probe::scan_child_bus_extend(child, subordinate - secondary).
  - if cmax > subordinate: WARN.
  - if subordinate > max: max = subordinate.
- else:
  - /* Pass 1: assign new bus number */
  - fixed_buses = pci_ea_fixed_busnrs(dev, &fixed_sec, &fixed_sub).
  - next_busnr = fixed_buses ? fixed_sec : max + 1.
  - child = Probe::add_new_bus(bus, dev, next_busnr).
  - max += 1.
  - /* Program PCI_PRIMARY_BUS = (primary | secondary << 8 | subordinate << 16) in single write */
  - config-write PCI_PRIMARY_BUS (composed u32).
  - child.bridge_ctl = bctl.
  - max = Probe::scan_child_bus_extend(child, available_buses).
  - if fixed_buses: max = fixed_sub.
  - pci_bus_update_busn_res_end(child, max).
  - config-write PCI_SUBORDINATE_BUS = max.
- /* Clear secondary status errors */
- config-write PCI_SEC_STATUS 0xffff.
- /* Restore bridge-control */
- config-write PCI_BRIDGE_CONTROL bctl.
- pm_runtime_put(dev).
- return max.

REQ-13: PciDev::cfg_space_size(dev):
- /* Default for type-1 bridge */
- if dev.hdr_type == PCI_HEADER_TYPE_NORMAL ∧ dev.class == PCI_CLASS_BRIDGE_HOST: return PCI_CFG_SPACE_SIZE.
- /* Type-1 (PCI-X / PCIe) probe: read 4-byte at 0x100 */
- if pos = pci_find_capability(dev, PCI_CAP_ID_PCIX) ∨ pci_is_pcie(dev):
  - status = config-read u32 at PCI_CFG_SPACE_SIZE=0x100.
  - if PCI_POSSIBLE_ERROR(status) ∨ pci_ext_cfg_is_aliased(dev): return PCI_CFG_SPACE_SIZE /* 256 */.
  - return PCI_CFG_SPACE_EXP_SIZE /* 4096 */.
- return PCI_CFG_SPACE_SIZE.

REQ-14: PciDev::init_capabilities — called after pci_device_add (from probe path):
- pci_ea_init(dev) /* Enhanced Allocation */.
- pci_msi_init(dev) /* per-MSI cap discover + disable */.
- pci_msix_init(dev) /* per-MSI-X cap discover + disable */.
- pci_allocate_cap_save_buffers(dev).
- pci_imm_ready_init(dev).
- pci_pm_init(dev) /* Power Management cap */.
- pci_vpd_init(dev) /* Vital Product Data */.
- pci_configure_ari(dev) /* per-ARI Forwarding Enable on parent */.
- pci_iov_init(dev) /* per-SR-IOV cap (extended) */.
- pci_ats_init(dev) /* per-ATS cap */.
- pci_pri_init(dev) /* per-PRI cap */.
- pci_pasid_init(dev) /* per-PASID cap */.
- pci_acs_init(dev) /* per-ACS cap */.
- pci_ptm_init(dev) /* per-PTM cap */.
- pci_aer_init(dev) /* per-AER cap */.
- pci_dpc_init(dev) /* per-DPC cap */.
- pci_rcec_init(dev) /* per-RCEC */.
- pci_doe_init(dev) /* per-DOE cap */.
- pci_tph_init(dev) /* per-TPH cap */.
- pci_rebar_init(dev) /* per-Resizable-BAR cap */.
- pci_dev3_init(dev) /* per-Device-3 cap */.
- pci_ide_init(dev) /* per-IDE cap */.
- pcie_report_downtraining(dev).
- pci_init_reset_methods(dev).

REQ-15: pci_msi_init(dev) / pci_msix_init(dev):
- dev.msi_cap = pci_find_capability(dev, PCI_CAP_ID_MSI).
- if dev.msi_cap:
  - ctrl = config-read u16 at msi_cap+PCI_MSI_FLAGS.
  - if ctrl & PCI_MSI_FLAGS_ENABLE: config-write to clear ENABLE (defense against firmware-left-enabled MSI).
  - if !(ctrl & PCI_MSI_FLAGS_64BIT): dev.msi_addr_mask = DMA_BIT_MASK(32).
- dev.msix_cap = pci_find_capability(dev, PCI_CAP_ID_MSIX).
- if dev.msix_cap:
  - ctrl = config-read u16 at msix_cap+PCI_MSIX_FLAGS.
  - if ctrl & PCI_MSIX_FLAGS_ENABLE: config-write to clear ENABLE.

REQ-16: Probe::device_add(dev, bus):
- pci_configure_device(dev) /* MPS, ExtTag, Relaxed-Ordering, LTR, ASPM-L1SS, EETLP, SERR, RCB */.
- device_initialize(&dev.dev).
- dev.dev.release = pci_release_dev.
- set_dev_node(&dev.dev, pcibus_to_node(bus)).
- dev.dev.dma_mask = &dev.dma_mask; coherent_dma_mask = 0xffffffffull; max_seg_size = 65536; seg_boundary = 0xffffffff.
- pcie_failed_link_retrain(dev).
- pci_fixup_device(pci_fixup_header, dev).
- pci_reassigndev_resource_alignment(dev).
- PciDev::init_capabilities(dev).
- /* Add to bus device list */
- down_write(&pci_bus_sem); list_add_tail(&dev.bus_list, &bus.devices); up_write.
- pcibios_device_add(dev).
- PciDev::set_msi_domain(dev) /* attach MSI irq_domain inherited from bus */.
- device_add(&dev.dev) /* triggers driver-bus match */.
- pci_tsm_init(dev); pci_npem_create(dev); pci_doe_sysfs_init(dev).

REQ-17: PciHostBridge::register / Probe::host_probe(bridge):
- pci_lock_rescan_remove.
- ret = Probe::scan_root_bus_bridge(bridge).
- pci_unlock_rescan_remove.
- if ret < 0: dev_err; return ret.
- bus = bridge.bus.
- if bridge.preserve_config: pci_bus_claim_resources(bus).
- pci_assign_unassigned_root_bus_resources(bus) /* cross-ref setup-bus.md */.
- for child in bus.children: pcie_bus_configure_settings(child).
- pci_lock_rescan_remove; pci_bus_add_devices(bus); pci_unlock_rescan_remove.
- pm_runtime_set_active / no_callbacks / devm_pm_runtime_enable on bridge.dev.
- return 0.

REQ-18: pci_scan_root_bus_bridge(bridge):
- /* Find IORESOURCE_BUS window in bridge.windows */
- found = false.
- resource_list_for_each_entry(window, &bridge.windows):
  - if window.res.flags & IORESOURCE_BUS: bridge.busnr = window.res.start; found = true; break.
- ret = pci_register_host_bridge(bridge); if ret < 0: return ret.
- b = bridge.bus; bus = bridge.busnr.
- if !found: pci_bus_insert_busn_res(b, bus, 255) /* default bus 0..255 */.
- max = Probe::scan_child_bus(b).
- if !found: pci_bus_update_busn_res_end(b, max).
- return 0.

## Acceptance Criteria

- [ ] AC-1: pci_scan_root_bus on a known QEMU q35 model produces upstream-baseline `lspci -D` byte-identical (vendor, device, class, BAR sizes, subsystem ids).
- [ ] AC-2: BAR sizing: write 0xFFFFFFFF → read-back → mask correctly decodes 4 KiB / 1 MiB / 64 MiB / 4 GiB / 8 GiB BARs (64-bit prefetchable) on NVMe + 10G NIC + GPU references.
- [ ] AC-3: 64-bit BAR straddling 4 GiB on 32-bit kernel marks IORESOURCE_UNSET | IORESOURCE_DISABLED (no garbage CPU mapping).
- [ ] AC-4: Bridge enumeration: primary / secondary / subordinate bus numbers assigned in a single PCI_PRIMARY_BUS write; cmax > subordinate triggers WARN.
- [ ] AC-5: ARI cap forwards device-number range 0..255 instead of 0..31; SR-IOV PF expands VF address space accordingly.
- [ ] AC-6: MFD bit on function-0 enables enumeration of functions 1..7; single-function device (MFD=0) skips funcs 1..7.
- [ ] AC-7: PCIe Downstream Port (only_one_child=1) skips devfn > 0 unless PCI_SCAN_ALL_PCIE_DEVS flag set.
- [ ] AC-8: PCIe port-type classification: ENDPOINT, LEG_END, ROOT_PORT, UPSTREAM, DOWNSTREAM, PCI_BRIDGE, PCIE_BRIDGE, RC_END, RC_EC all decoded from PCI_EXP_FLAGS_TYPE.
- [ ] AC-9: Misreported downstream-of-downstream / upstream-of-upstream auto-corrected by set_pcie_port_type.
- [ ] AC-10: pci_msi_init / pci_msix_init: discovers MSI/MSI-X cap offset; clears ENABLE bit if firmware-left-on; MSI 32-bit caps narrow msi_addr_mask to DMA_BIT_MASK(32).
- [ ] AC-11: pci_init_capabilities walks std + ext cap lists and initializes every supported per-cap state exactly once; cycle in cap-list → WARN + return (no infinite loop).
- [ ] AC-12: Configuration Request Retry Status (RRS) handled: exponential-backoff retry up to 60 s before giving up with WARN.
- [ ] AC-13: Extended config space: device with 4 KiB cfg → cfg_size == 4096; aliased ext config (ASM1083 / SBx00 quirk class) → cfg_size == 256.
- [ ] AC-14: Hotplug rescan via pci_rescan_bus: new devices added without disturbing already-bound ones; pci_lock_rescan_remove serializes against pci_remove_bus_device.
- [ ] AC-15: Multi-domain (multi-PCI-segment) system: each segment has independent bus-number space, displayed as `<segment>:<bus>:<dev>.<func>`.

## Architecture

```
struct PciDev {
  bus: *PciBus,
  devfn: u8,
  vendor: u16,
  device: u16,
  subsystem_vendor: u16,
  subsystem_device: u16,
  class: u32,                       // 24-bit, in upper 3 bytes
  revision: u8,
  hdr_type: u8,                     // 0/1/2 normal/bridge/cardbus
  multifunction: bool,
  cfg_size: u16,                    // 256 or 4096
  pcie_cap: u8,
  msi_cap: u8,
  msix_cap: u8,
  pcie_flags_reg: u16,              // PCI_EXP_FLAGS at pcie_cap+2
  pcie_mpss: u8,                    // Max Payload Size Supported (0..5)
  supported_speeds: u32,            // bitmap of supported PCIe gen
  link_active_reporting: bool,      // LNKCAP DLLLARC
  aspm_l0s_support: bool,
  aspm_l1_support: bool,
  is_hotplug_bridge: bool,
  is_pciehp: bool,
  is_thunderbolt: bool,
  is_cxl: bool,
  untrusted: bool,
  non_compliant_bars: bool,
  mmio_always_on: bool,
  broken_intx_masking: bool,
  error_state: PciChannelState,     // pci_channel_io_normal at probe
  dma_mask: u32,                    // 0xffffffff default (32-bit)
  msi_addr_mask: u64,               // DMA_BIT_MASK(64) default
  current_state: u8,                // PCI_UNKNOWN at probe
  resource: [Resource; PCI_NUM_RESOURCES],   // BARs + ROM + bridge windows
  bus_list: ListNode,
  pcie_cap_lock: SpinLock,
  msi_lock: RawSpinLock,
}

struct PciBus {
  parent: Option<*PciBus>,
  number: u8,
  primary: u8,                      // upstream bus
  busn_res: Resource,               // bus-number range
  devices: List<PciDev>,
  children: List<PciBus>,
  self: Option<*PciDev>,            // upstream bridge dev, None for root
  bridge: *Device,
  ops: *PciOps,
  bus_flags: u32,                   // PCI_BUS_FLAGS_*
  bridge_ctl: u16,
  is_added: bool,
  name: [u8; 48],
}

struct PciHostBridge {
  dev: Device,
  bus: *PciBus,
  busnr: u8,
  windows: List<ResourceEntry>,
  sysdata: *Void,
  ops: *PciOps,
  preserve_config: bool,
  msi_domain: Option<*IrqDomain>,
}
```

`Probe::scan_root_bus_bridge(bridge) -> Result<()>`:
1. /* Find bus-number resource window */
2. found = false.
3. for window in bridge.windows:
   - if window.res.flags & IORESOURCE_BUS: bridge.busnr = window.res.start; found = true; break.
4. PciHostBridge::register(bridge)?.
5. b = bridge.bus; bus = bridge.busnr.
6. if !found: PciBus::insert_busn_res(b, bus, 255) /* default bus 0..255 */.
7. max = Probe::scan_child_bus(b).
8. if !found: PciBus::update_busn_res_end(b, max).
9. Ok.

`Probe::scan_child_bus_extend(bus, avail) -> u32`:
1. for devnr in 0..PCI_MAX_NR_DEVS=32:
   - Probe::scan_slot(bus, PCI_DEVFN(devnr, 0)).
2. used = pci_iov_bus_range(bus); max += used.
3. if !bus.is_added: pcibios_fixup_bus(bus); bus.is_added = true.
4. /* Count bridges */
5. for_each_pci_bridge(dev, bus):
   - if dev.is_hotplug_bridge: hotplug_bridges += 1.
   - else: normal_bridges += 1.
6. /* Pass-0: scan pre-configured bridges */
7. for_each_pci_bridge(dev, bus):
   - max = Probe::scan_bridge_extend(bus, dev, max, 0, 0).
8. /* Pass-1: scan bridges needing reconfiguration */
9. for_each_pci_bridge(dev, bus):
   - distribute available_buses among hotplug bridges.
   - max = Probe::scan_bridge_extend(bus, dev, cmax, buses, 1).
10. if bus.self ∧ bus.self.is_hotplug_bridge: reserve pci_hotplug_bus_size buses.
11. return max.

`Probe::scan_slot(bus, devfn) -> u32`:
1. if only_one_child(bus) ∧ devfn > 0: return 0.
2. fn = 0; nr = 0.
3. loop:
   - dev = Probe::scan_single_device(bus, devfn + fn).
   - if dev.is_some():
     - if !pci_dev_is_added(dev): nr += 1.
     - if fn > 0: dev.multifunction = 1.
   - else if fn == 0:
     - if !hypervisor_isolated_pci_functions(): break.
   - fn = next_fn(bus, dev, fn).
   - if fn < 0: break.
4. if bus.self ∧ nr > 0: pcie_aspm_init_link_state(bus.self).
5. return nr.

`Probe::scan_single_device(bus, devfn) -> Option<PciDev>`:
1. /* Already added? */
2. existing = pci_get_slot(bus, devfn).
3. if existing.is_some(): pci_dev_put(existing); return Some(existing).
4. /* Read vendor/device w/ RRS retry */
5. let mut l: u32; if !PciBus::read_dev_vendor_id(bus, devfn, &mut l, 60_000): return None.
6. dev = PciDev::alloc(bus)?.
7. dev.devfn = devfn; dev.vendor = (l & 0xffff) as u16; dev.device = ((l >> 16) & 0xffff) as u16.
8. if PciDev::setup(dev).is_err(): drop dev; return None.
9. Probe::device_add(dev, bus).
10. Some(dev).

`PciDev::setup(dev) -> Result<()>`:
1. hdr_type_byte = config_read_u8(dev, PCI_HEADER_TYPE).
2. dev.hdr_type = hdr_type_byte & PCI_HEADER_TYPE_MASK.
3. dev.multifunction = (hdr_type_byte & PCI_HEADER_TYPE_MFD) != 0.
4. /* device-model wiring */
5. dev.dev.bus = pci_bus_type; dev.dev.parent = bus.bridge; dev.error_state = pci_channel_io_normal.
6. PciDev::set_pcie_port_type(dev).
7. dev.dma_mask = 0xffffffff; dev.msi_addr_mask = DMA_BIT_MASK(64).
8. dev_set_name(&dev.dev, "%04x:%02x:%02x.%d", domain, bus.number, SLOT, FUNC).
9. /* Class + cfg-size */
10. class = config_read_u32(dev, PCI_CLASS_REVISION).
11. dev.revision = (class & 0xff) as u8; dev.class = class >> 8.
12. dev.cfg_size = PciDev::cfg_space_size(dev).
13. /* Misc port-type / flags */
14. set_pcie_thunderbolt(dev); set_pcie_cxl(dev); set_pcie_untrusted(dev).
15. if pci_is_pcie(dev): dev.supported_speeds = pcie_get_supported_speeds(dev).
16. dev.current_state = PCI_UNKNOWN.
17. pci_fixup_device(pci_fixup_early, dev); pci_set_removable(dev).
18. /* Header-type dispatch */
19. match dev.hdr_type:
    - PCI_HEADER_TYPE_NORMAL:
      - if class >> 8 == PCI_CLASS_BRIDGE_PCI: bad_class WARN; class = PCI_CLASS_NOT_DEFINED << 8.
      - pci_read_irq(dev).
      - pci_read_bases(dev, PCI_STD_NUM_BARS, PCI_ROM_ADDRESS).
      - pci_subsystem_ids(dev, &dev.subsystem_vendor, &dev.subsystem_device).
    - PCI_HEADER_TYPE_BRIDGE:
      - pci_read_irq(dev). dev.transparent = (dev.class & 0xff) == 1.
      - pci_read_bases(dev, 2, PCI_ROM_ADDRESS1).
      - pci_read_bridge_windows(dev).
      - PciDev::set_pcie_hotplug(dev).
      - SSVID/SSID via PCI_CAP_ID_SSVID lookup.
    - PCI_HEADER_TYPE_CARDBUS:
      - if class >> 8 != PCI_CLASS_BRIDGE_CARDBUS: bad_class.
      - pci_read_irq(dev); pci_read_bases(dev, 1, 0); CardBus SSVID/SSID.
    - default: ERR "unknown header type"; return Err(EIO).
20. Ok.

`PciDev::set_pcie_port_type(pdev)`:
1. pos = pci_find_capability(pdev, PCI_CAP_ID_EXP).
2. if pos == 0: return.
3. pdev.pcie_cap = pos.
4. pdev.pcie_flags_reg = config_read_u16(pdev, pos + PCI_EXP_FLAGS).
5. type = FIELD_GET(PCI_EXP_FLAGS_TYPE, pdev.pcie_flags_reg).
6. if type == PCI_EXP_TYPE_ROOT_PORT: pci_enable_rrs_sv(pdev).
7. pdev.devcap = config_read_u32(pdev, pos + PCI_EXP_DEVCAP).
8. pdev.pcie_mpss = FIELD_GET(PCI_EXP_DEVCAP_PAYLOAD, pdev.devcap).
9. linkcap = config_read_u32(pdev, pos + PCI_EXP_LNKCAP).
10. pdev.link_active_reporting = (linkcap & PCI_EXP_LNKCAP_DLLLARC) != 0.
11. /* ASPM (under CONFIG_PCIEASPM) */
12. pdev.aspm_l0s_support = (linkcap & PCI_EXP_LNKCAP_ASPM_L0S) != 0.
13. pdev.aspm_l1_support = (linkcap & PCI_EXP_LNKCAP_ASPM_L1) != 0.
14. parent = pci_upstream_bridge(pdev).
15. /* Auto-correct misreported port type */
16. if parent.is_some():
    - if type == DOWNSTREAM ∧ pcie_downstream_port(parent): set to UPSTREAM (warn).
    - else if type == UPSTREAM ∧ pci_pcie_type(parent) == UPSTREAM: set to DOWNSTREAM (warn).

`PciDev::init_capabilities(dev)`:
1. pci_ea_init(dev) /* Enhanced Allocation */.
2. PciDev::msi_init(dev) /* discover + disable MSI */.
3. PciDev::msix_init(dev) /* discover + disable MSI-X */.
4. pci_allocate_cap_save_buffers(dev).
5. pci_imm_ready_init(dev); pci_pm_init(dev); pci_vpd_init(dev).
6. /* Per-cap discovery + state init */
7. pci_configure_ari(dev) /* per-ARI Forwarding Enable on parent */.
8. pci_iov_init(dev); pci_ats_init(dev); pci_pri_init(dev); pci_pasid_init(dev).
9. pci_acs_init(dev); pci_ptm_init(dev); pci_aer_init(dev); pci_dpc_init(dev).
10. pci_rcec_init(dev); pci_doe_init(dev); pci_tph_init(dev); pci_rebar_init(dev).
11. pci_dev3_init(dev); pci_ide_init(dev).
12. pcie_report_downtraining(dev); pci_init_reset_methods(dev).

`Probe::scan_bridge_extend(bus, dev, max, available_buses, pass) -> u32`:
1. pm_runtime_get_sync(dev).
2. buses = config_read_u32(dev, PCI_PRIMARY_BUS).
3. primary, secondary, subordinate ← FIELD_GET on buses.
4. /* Hardware glitch: hardwired-to-0 primary */
5. if primary == 0 ∧ primary != bus.number ∧ secondary > 0 ∧ subordinate > 0: primary = bus.number.
6. /* Validate */
7. if !pass ∧ (primary != bus.number ∨ secondary <= bus.number ∨ secondary > subordinate): broken = 1.
8. bctl = config_read_u16(dev, PCI_BRIDGE_CONTROL).
9. /* Disable master-abort during probe */
10. config_write_u16(dev, PCI_BRIDGE_CONTROL, bctl & ~PCI_BRIDGE_CTL_MASTER_ABORT).
11. if pci_is_cardbus_bridge(dev): max = pci_cardbus_scan_bridge_extend(...); goto out.
12. /* Branch on pre-configured vs fresh */
13. if (secondary ∨ subordinate) ∧ !broken:
    - if pass: goto out /* skip in pass-1 */.
    - child = pci_find_bus(domain, secondary).is_some() ? : Probe::add_new_bus(bus, dev, secondary).
    - cmax = Probe::scan_child_bus_extend(child, subordinate - secondary).
    - if cmax > subordinate: WARN.
14. else /* assign new */:
    - if !pass: goto out (assign in pass-1).
    - fixed = pci_ea_fixed_busnrs(dev, &fs, &fb); next_busnr = fixed ? fs : max + 1.
    - child = Probe::add_new_bus(bus, dev, next_busnr); max += 1.
    - /* Single PCI_PRIMARY_BUS write programs primary/secondary/subordinate */
    - composed = (primary | child.busn_res.start << 8 | child.busn_res.end << 16).
    - config_write_u32(dev, PCI_PRIMARY_BUS, composed).
    - max = Probe::scan_child_bus_extend(child, available_buses).
    - if fixed: max = fb.
    - PciBus::update_busn_res_end(child, max).
    - config_write_u8(dev, PCI_SUBORDINATE_BUS, max).
15. /* Clear secondary status */
16. config_write_u16(dev, PCI_SEC_STATUS, 0xffff).
17. config_write_u16(dev, PCI_BRIDGE_CONTROL, bctl) /* restore */.
18. pm_runtime_put(dev).
19. return max.

`Probe::device_add(dev, bus)`:
1. pci_configure_device(dev) /* MPS, ExtTag, LTR, ASPM, EETLP, SERR, RCB, ACPI HP-params */.
2. device_initialize(&dev.dev).
3. dev.dev.release = pci_release_dev.
4. set_dev_node(&dev.dev, pcibus_to_node(bus)).
5. dev.dev.dma_mask = &dev.dma_mask; coherent_dma_mask = 0xffffffffull.
6. dma_set_max_seg_size(&dev.dev, 65536); dma_set_seg_boundary(&dev.dev, 0xffffffff).
7. pcie_failed_link_retrain(dev).
8. pci_fixup_device(pci_fixup_header, dev).
9. pci_reassigndev_resource_alignment(dev).
10. PciDev::init_capabilities(dev).
11. down_write(&pci_bus_sem); list_add_tail(&dev.bus_list, &bus.devices); up_write.
12. pcibios_device_add(dev).
13. PciDev::set_msi_domain(dev).
14. device_add(&dev.dev).
15. pci_tsm_init(dev); pci_npem_create(dev); pci_doe_sysfs_init(dev).

`PciDev::release(dev)` /* called when refcount drops to 0 */:
1. pci_release_capabilities(dev) /* AER exit, RCEC exit, IOV release, free cap-save buffers */.
2. pci_release_of_node(dev); pcibios_release_device(dev).
3. pci_bus_put(dev.bus).
4. bitmap_free(dev.dma_alias_mask).
5. kfree(dev).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_list_bounded` | TERMINATION | per-pci_find_capability: std-cap walk ≤ 48 iters; per-pci_find_ext_capability: ext-cap walk ≤ 480 iters; cycle ⟹ WARN + return 0. |
| `bar_size_no_overflow` | OVERFLOW | per-`__pci_read_base`: pci_size(l64, sz64, mask64) does not overflow u64. |
| `devfn_iteration_bounded` | TERMINATION | per-scan_slot loop: next_fn strictly increasing or returns -ENODEV (no infinite loop on malformed ARI). |
| `pci_dev_no_uaf` | UAF | per-pci_release_dev: refcount==0 enforced before kfree; dev.bus reference put. |
| `pci_bus_no_uaf` | UAF | per-pci_create_root_bus / pci_add_new_bus: refcount get on parent bus / bridge. |
| `bridge_bus_numbers_consistent` | INVARIANT | per-scan_bridge_extend: child.busn_res.start ≥ bus.number + 1 ∧ child.busn_res.end ≤ bus.busn_res.end. |
| `rrs_wait_bounded` | TERMINATION | per-pci_bus_wait_rrs: total delay sum ≤ 60 s; gives up with WARN on timeout. |
| `port_type_correction_idempotent` | INVARIANT | per-set_pcie_port_type: at most one type swap (UPSTREAM↔DOWNSTREAM) per device. |

### Layer 2: TLA+

`models/pci/probe_recursion.tla` (parent-declared): proves recursive bus enumeration terminates.

`models/pci/cap_iteration.tla` (parent-declared): proves std + ext cap iteration always terminates; cycle ⟹ WARN.

`models/pci/bridge_bus_assignment.tla`:
- States: bus-number free-list, child-bus tree.
- Properties:
  - `safety_no_bus_overlap` — per-scan_bridge: assigned bus ranges are non-overlapping.
  - `safety_subordinate_geq_secondary` — per-bridge: subordinate ≥ secondary.
  - `safety_subordinate_leq_parent_end` — per-bridge: subordinate ≤ parent.busn_res.end.
  - `liveness_scan_terminates` — per-scan_child_bus: returns finite max.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Probe::scan_single_device` post: Some(dev) ⟹ dev.bus == bus ∧ dev.devfn == devfn ∧ dev.vendor != 0xFFFF ∧ dev.device != 0xFFFF | `Probe::scan_single_device` |
| `PciDev::setup` post: dev.hdr_type ∈ {NORMAL, BRIDGE, CARDBUS}; otherwise Err(EIO) | `PciDev::setup` |
| `PciDev::read_base` post: res.flags either valid (IO ∨ MEM, ±MEM_64, ±PREFETCH, +SIZEALIGN) or 0 (BAR unimplemented) | `PciDev::read_base` |
| `PciDev::cfg_space_size` post: returns 256 ∨ 4096 | `PciDev::cfg_space_size` |
| `PciDev::init_capabilities` post: msi_cap, msix_cap, pcie_cap all set if present in config space, else 0 | `PciDev::init_capabilities` |
| `PciDev::msi_init` post: dev.msi_cap == 0 ∨ PCI_MSI_FLAGS_ENABLE cleared in config | `PciDev::msi_init` |
| `Probe::scan_bridge_extend` post: returned max ≥ bus.number; child bus added to parent.children | `Probe::scan_bridge_extend` |
| `Probe::device_add` post: dev added to bus.devices; sysfs entry created; MSI domain attached | `Probe::device_add` |

### Layer 4: Verus/Creusot functional

`Probe::scan_root_bus(parent, bus_nr, ops, sysdata, resources)` ↔ upstream `pci_scan_root_bus` semantic equivalence: for any reference HW topology with vendor-id-readable devices, the produced `pci_bus.devices` list (vendor, device, devfn, class, BARs, subsystem ids) matches upstream byte-for-byte in iteration order. Encoded as Verus refinement: `forall hw_topology. rookery_scan(hw_topology) == upstream_scan(hw_topology)`.

## Hardening

(Inherits row-1 features from `drivers/pci/00-overview.md` § Hardening.)

probe-specific reinforcement:

- **Capability list walk bounded** — std caps ≤ 48 iters, ext caps ≤ 480 iters; cycle detected via visited-offsets bitmap (defense against per-malformed-device producing infinite cap-list loop). Layer-2 `cap_iteration.tla` is the proof.
- **BAR sizing under PCI_COMMAND-decode-disabled** — IO/MEM decode bits in PCI_COMMAND temporarily cleared during write-0xFF / read-back BAR sizing (defense against per-in-flight-DMA seeing transient 0xFFFFFFFF write).
- **Bridge master-abort disabled during probe** — PCI_BRIDGE_CTL_MASTER_ABORT cleared in PCI_BRIDGE_CONTROL during scan_bridge_extend (defense against per-spurious-AER on transient empty config-space).
- **devfn iteration policy by ARI presence** — without ARI, only 0..7 funcs per slot scanned; with ARI on parent bridge, 0..255 devfns scanned (per-spec); malformed ARI list with next_fn ≤ fn returns -ENODEV (defense against infinite loop).
- **MFD bit checked on function 0** before scanning funcs 1..7 (defense against per-single-function-device with stale data at funcs 1..7 causing ghost-device enumeration).
- **RRS wait bounded to 60 s** with exponential backoff and per-second WARN (defense against per-broken-device-RRS-loop hanging boot).
- **Aliased extended config detected** — per-`pci_ext_cfg_is_aliased`: every 256-byte alias compared to header; aliased ⟹ cfg_size capped at 256 (defense against per-buggy-bridge misforwarding ext-config).
- **PCI_PRIMARY_BUS programmed in single dword write** — primary/secondary/subordinate written atomically to avoid transient unreachable bus state visible to other CPUs (defense against per-config-cycle race during scan).
- **MSI and MSI-X ENABLE bits cleared on probe** — firmware-left-enabled MSI/MSI-X disabled during pci_msi_init / pci_msix_init (defense against per-pre-driver-init MSI floods).
- **MasterAbort + Secondary-status cleared post-scan** — `PCI_SEC_STATUS = 0xffff` and bctl restored at end of scan_bridge_extend (defense against per-scan-residual-error log).
- **pci_lock_rescan_remove mutex** — serializes pci_rescan_bus + pci_remove_bus_device + pci_stop_and_remove_bus_device (defense against per-concurrent-rescan tearing pci_bus.devices list).
- **only_one_child PCIe optimization** — PCIe Downstream Ports scan only Device 0 on the downstream link unless PCI_SCAN_ALL_PCIE_DEVS flag set (per PCIe spec r3.1, sec 7.3.1; defense against per-bogus-devfn-1..31 ghost enumeration).
- **Port-type auto-correction logged** — when set_pcie_port_type detects misreported UPSTREAM↔DOWNSTREAM via parent topology, emits pci_info + corrects in pcie_flags_reg cache (defense against per-driver-binding-wrong-role).

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — /sys/bus/pci/devices/<dev>/resource[N], /sys/bus/pci/rescan, hotplug-slot sysfs read buffers whitelisted; defense against per-oversized-sysfs-read leaking adjacent slab.
- **PAX_KERNEXEC** — pci_scan_bus, pci_scan_slot, pci_alloc_dev, pci_setup_device, __pci_read_base run W^X.
- **PAX_RANDKSTACK** — per-/sys/bus/pci/rescan-write and per-pci_scan_bridge recursion entry randomize kernel-stack offset.
- **PAX_REFCOUNT** — struct pci_dev, struct pci_bus, struct pci_host_bridge, pci_slot refs saturating refcount_t; defense against per-hotplug-storm refcount overflow.
- **PAX_MEMORY_SANITIZE** — pci_dev, pci_bus, pci_saved_state slabs poison-on-free; defense against per-prior-device-state leak across hot-unplug/replug into a sibling slot.
- **PAX_UDEREF** — /sys/bus/pci/{rescan,remove} enforce split user/kernel on the bool/int parsed from userspace.
- **PAX_RAP/kCFI** — pci_host_bridge_ops, pci_ops, struct pci_driver probe/remove/shutdown vtables CFI-protected.
- **GRKERNSEC_HIDESYM** — pci_root_buses, per-host-bridge ops, pci_scan_root_bus_bridge addresses hidden from kallsyms.
- **GRKERNSEC_DMESG** — "pci %s: [%04x:%04x]", "pci_bus %s: scanning bus" prints restricted; defense against per-topology-disclosure.
- **BAR sizing CAP_SYS_RAWIO** — pci_read_bases / __pci_read_base BAR-sizing write of 0xFFFFFFFF and re-read is a kernel-internal operation triggered by probe; the userspace counterpart (/sys/bus/pci/devices/<dev>/resource[N]_resize and PCIIOC_*) gated on CAP_SYS_RAWIO; defense against per-unprivileged BAR-window manipulation.
- **Hotplug audit** — /sys/bus/pci/rescan, /sys/bus/pci/devices/<dev>/remove, /sys/bus/pci/devices/<dev>/reset, and ACPI/PCIe-NHP slot enable/disable events audited (LSM + ratelimited dmesg); defense against per-rescan-storm being weaponized for DoS or per-rogue-device hot-add evading detection.
- **PCI rescan / remove CAP_SYS_ADMIN** — /sys/bus/pci/rescan, /sys/bus/pci/devices/<dev>/remove, /sys/bus/pci/devices/<dev>/reset writers require CAP_SYS_ADMIN.
- **only_one_child PCIe enforcement** — Downstream Ports scan only Device 0 unless PCI_SCAN_ALL_PCIE_DEVS flag set; defense against per-bogus-devfn-1..31 ghost enumeration.
- **Per-pci_alloc_dev kmemleak object** — kmemleak-tracked alloc/free so a hot-add/hot-remove leak is observable.
- Rationale: PCI probe is the device-attachment plane that turns wire-level PCIe topology into kernel objects bound to drivers; grsec posture combines CAP_SYS_ADMIN on rescan/remove/reset, CAP_SYS_RAWIO on BAR/resource-resize, audit on every hot-add/remove event, CFI on host-bridge/driver vtables, and sanitize-on-free of pci_dev/saved-state slabs.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-cap implementation details (covered in `msi.md`, `pcie-aer.md`, `iov.md`, `ats.md`, `pcie-dpc.md`, `acs.md` future Tier-3s)
- BAR-window assignment + reassignment (covered in `setup-bus.md` + `setup-res.md` future Tier-3s)
- Config-space accessor backends (covered in `access.md`)
- ACPI MCFG parsing + per-segment domain (covered in `pci-acpi.md` future Tier-3)
- pci_driver bind / unbind / probe callbacks (covered in `pci-driver.md` future Tier-3)
- Quirks + per-device fixups (covered in `quirks.md` future Tier-3)
- Hot-plug controllers (covered in `hotplug-pciehp.md` + `hotplug-acpiphp.md` future Tier-3s)
- pcie_aspm_init_link_state + ASPM state machine (covered in `pcie-aspm.md` future Tier-3)
- CardBus enumeration (legacy; deferred)
- 32-bit-only paths
- Implementation code
