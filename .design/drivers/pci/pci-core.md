# Tier-3: drivers/pci/pci.c — PCI/PCIe subsystem-core API surface

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/pci/00-overview.md
upstream-paths:
  - drivers/pci/pci.c (~6800 lines)
  - drivers/pci/pci.h
  - include/linux/pci.h
  - include/linux/pci_regs.h (UAPI side: include/uapi/linux/pci_regs.h)
  - include/linux/pci_ids.h
  - Documentation/PCI/pci.rst
-->

## Summary

`drivers/pci/pci.c` is the **subsystem-core API file** of the PCI/PCIe stack. It is the home of the most heavily exported PCI API surface — the routines every PCI driver, the PCI portdrv stack (AER/DPC/PME/PTM/bwctrl/ASPM), the IOMMU layer (`drivers/iommu/dma-iommu.c`), the IRQ subsystem (`msi/`), and arch-specific code call into. It does **not** itself enumerate the bus (probe.c), nor drive the bus_type matching (pci-driver.c), nor implement config-space transport (access.c); instead it implements:

- **Capability walking** — standard PCI cap list (`pci_find_capability`, `pci_find_next_capability`, `pci_find_ht_capability`, `pci_find_vsec_capability`), PCIe extended cap list (`pci_find_ext_capability`, `pci_find_next_ext_capability`, `pci_find_dvsec_capability`), DSN reader (`pci_get_dsn`).
- **BAR (Base Address Register) resource helpers** — `pci_find_parent_resource`, `pci_find_resource`, `pci_resource_name`, `pci_select_bars`, `pci_request_region(s)` / `pci_release_region(s)` (+ exclusive + selected variants), `pci_remap_iospace` / `pci_unmap_iospace`, `pci_ioremap_bar` / `pci_ioremap_wc_bar`, `pci_register_io_range` / `pci_pio_to_address` / `pci_address_to_pio`.
- **Enhanced Allocation (EA) cap parser** — `pci_ea_init`, `pci_ea_read`, `pci_ea_get_resource`, `pci_ea_flags` (legacy hardware-described BAR layout via EA capability instead of BAR sizing).
- **Power-management state machine** — `pci_set_power_state` / `_locked`, `pci_power_up`, `pci_set_full_power_state`, `pci_set_low_power_state`, `pci_update_current_state`, `pci_refresh_power_state`, `pci_platform_power_transition`, `pci_choose_state`, `pci_resume_bus`, `pci_bus_set_current_state`, `pci_pm_init`, `pci_prepare_to_sleep`, `pci_back_from_sleep`, `pci_finish_runtime_suspend`, plus PME family (`pci_pme_capable`, `pci_pme_active`, `pci_pme_restore`, `pci_pme_wakeup`, `pci_pme_wakeup_bus`, `pci_pme_list_scan`, `pci_check_pme_status`, `pcie_clear_root_pme_status`) and wake (`pci_enable_wake`, `pci_wake_from_d3`, `pci_dev_run_wake`, `pci_dev_need_resume`, `pci_dev_adjust_pme`, `pci_dev_complete_resume`).
- **D3cold policy** — `pci_bridge_d3_possible`, `pci_bridge_d3_update`, `pci_d3cold_enable` / `_disable`, `pci_dev_check_d3cold`, `bridge_d3_blacklist` (DMI quirks).
- **Saved-state machinery** — `pci_save_state`, `pci_restore_state`, `pci_restore_config_space`, `pci_store_saved_state`, `pci_load_saved_state`, `pci_load_and_free_saved_state`, `pci_find_saved_cap`, `pci_find_saved_ext_cap`, `pci_add_cap_save_buffer`, `pci_add_ext_cap_save_buffer`, `pci_allocate_cap_save_buffers`, `pci_free_cap_save_buffers`, PCIe state save/restore (`pci_save_pcie_state` / `_restore_`), PCI-X (`pci_save_pcix_state` / `_restore_`). Saved state is the bedrock of D3hot → D0 resume, FLR, hot-reset, and SRIOV-VF re-init.
- **Device enable / disable** — `pci_enable_device`, `pci_enable_device_mem`, `pci_enable_device_flags`, `pci_reenable_device`, `pci_disable_device`, `pci_disable_enabled_device`, `do_pci_enable_device`, `pci_enable_bridge`, `pci_host_bridge_enable_device` / `_disable_device`, `pci_set_master`, `pci_clear_master`, `__pci_set_master` (drives `PCI_COMMAND_MASTER` / `_MEM` / `_IO` bits, `enable_cnt` recursive counter, upstream bridge enable, INTx pin un-mask, ASPM-link config, arch `pcibios_enable_device` / `_set_master` weak hooks).
- **Memory-Write-Invalidate + cacheline** — `pci_set_mwi`, `pci_try_set_mwi`, `pci_clear_mwi`, `pci_set_cacheline_size` (legacy PCI).
- **INTx / parity** — `pci_intx`, `pci_disable_parity`.
- **ACS (Access Control Services)** — `pci_acs_init`, `pci_enable_acs`, `pci_std_enable_acs`, `pci_acs_enabled`, `pci_acs_path_enabled`, `pci_acs_flags_enabled`, `pci_request_acs`, `__pci_config_acs` (parses `pci=config_acs=...` and `disable_acs_redir=...` command line).
- **ARI (Alternative Routing-ID Interpretation)** — `pci_configure_ari` (per-bridge ARI-Forwarding-Enable in `PCI_EXP_DEVCTL2`).
- **ATS / AtomicOps** — `pci_enable_atomic_ops_to_root` (negotiates 32/64/128-bit atomic-op egress through every upstream bridge of the tree).
- **PCIe link / topology** — `pcie_link_speed_mbps`, `pcie_bandwidth_available`, `pcie_bandwidth_capable`, `pcie_get_supported_speeds`, `pcie_get_speed_cap`, `pcie_get_width_cap`, `pcie_get_mps` / `pcie_set_mps`, `pcie_get_readrq` / `pcie_set_readrq`, `pcie_print_link_status`, `__pcie_print_link_status`, `pcie_wait_for_link`, `pcie_wait_for_link_delay`, `pcie_wait_for_link_status`, `pcie_retrain_link`, `pcie_clear_device_status`. PCI-X: `pcix_get_mmrbc`, `pcix_get_max_mmrbc`, `pcix_set_mmrbc`.
- **Reset paths** — `pci_reset_function`, `pci_reset_function_locked`, `pci_try_reset_function`, `__pci_reset_function_locked`, `pci_init_reset_methods`, `pci_reset_bus`, `pci_try_reset_bus`, `pci_probe_reset_bus`, `pci_reset_bridge`, `pci_try_reset_bridge`, `pci_bus_error_reset`, `pcie_flr`, `pcie_reset_flr`, `pci_af_flr`, `pci_pm_reset`, `pci_bus_reset`, `pci_slot_reset`, `pci_try_reset_slot`, `pci_probe_reset_slot`, `pci_dev_reset_slot_function`, `pci_reset_hotplug_slot`, `pci_parent_bus_reset`, `pci_reset_bus_function`, `cxl_reset_bus_function`, `pci_bridge_secondary_bus_reset`, `pci_reset_secondary_bus`, `pci_bridge_wait_for_secondary_bus`. Per-device + per-bus + per-slot lock helpers (`pci_dev_lock`, `pci_dev_trylock`, `pci_dev_unlock`, `pci_bus_lock`, `pci_bus_unlock`, `pci_bus_trylock`, `pci_slot_lock`, `pci_slot_unlock`, `pci_slot_trylock`).
- **DMA-alias + presence** — `pci_add_dma_alias`, `pci_devs_are_dma_aliases`, `pci_device_is_present`, `pci_ignore_hotplug`, `pci_real_dma_dev` (weak), `pci_resource_to_user` (weak), `pci_pr3_present`.
- **VGA arbiter glue** — `pci_set_vga_state`, `pci_register_set_vga_state`, `pci_set_vga_state_arch`.
- **Domain numbering** — `pci_bus_find_domain_nr`, `pci_bus_release_domain_nr`, `pci_bus_find_emul_domain_nr`, `pci_bus_release_emul_domain_nr`, static + dynamic IDA, OF integration.
- **Resource-alignment overrides** — `pci_specified_resource_alignment`, `pci_request_resource_alignment`, `pci_reassigndev_resource_alignment`, `resource_alignment_show` / `_store`, `pci_resource_alignment_sysfs_init` (drives `pci=resource_alignment=...` reassignment for VFIO mediation).
- **`pci=` command-line parser** — `pci_setup`, `pci_realloc_setup_params`, `pcie_port_pm_setup`.

It also keeps the **PME poll workqueue** (`pci_pme_list_scan` on `system_freezable_wq`), the **ACS rule-spec parser**, and `__pcie_print_link_status` (the `[X.X GT/s PCIe x Y link at ...]` boot-time annotation every NIC/NVMe/GPU prints).

Critical for: every PCI driver bring-up (`pci_enable_device` + `pci_set_master`), every D3hot/D0 resume (`pci_save_state` / `_restore_state`), every reset path (FLR / hot-reset / bus-reset for `vfio-pci`), every IOMMU pairing (`pci_dev->dma_alias`, `pci_get_dsn` for CXL CMA), every MSI-X programming (saved-state replay), every userspace `setpci` / `lspci` / `vfio` interaction.

This Tier-3 covers `drivers/pci/pci.c` (~6800 lines). Bus enumeration (`probe.c`), driver matching (`pci-driver.c`), config-space transport (`access.c`), MSI/MSI-X allocation (`msi/`), AER body (`pcie/aer.c`), SR-IOV body (`iov.c`), ATS/PRI/PASID body (`ats.c`), bus.c helpers, setup-bus.c / setup-res.c assignment, and quirks.c are covered by sibling Tier-3 docs. This file owns the **API surface** they all hang off.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pci_dev` (in `<linux/pci.h>`) | per-device representation | `PciDev` |
| `struct pci_bus` | per-bus representation | `PciBus` |
| `struct pci_host_bridge` | per-root-complex bridge | `PciHostBridge` |
| `struct pci_acs` | per-walk ACS ctrl + fw_ctrl | `PciAcs` |
| `struct pci_saved_state` | per-device saved-config blob | `PciSavedState` |
| `struct pci_cap_saved_state` | per-cap saved blob | `PciCapSavedState` |
| `struct pci_pme_device` | per-PME-poll-list entry | `PciPmeDevice` |
| `pci_find_capability()` | per-std-cap walk entry | `PciCore::find_capability` |
| `pci_find_next_capability()` | per-std-cap walk iter | `PciCore::find_next_capability` |
| `pci_find_ext_capability()` | per-ext-cap walk entry | `PciCore::find_ext_capability` |
| `pci_find_next_ext_capability()` | per-ext-cap walk iter | `PciCore::find_next_ext_capability` |
| `pci_find_ht_capability()` | per-HT-cap walk | `PciCore::find_ht_capability` |
| `pci_find_vsec_capability()` | per-VSEC walk | `PciCore::find_vsec_capability` |
| `pci_find_dvsec_capability()` | per-DVSEC walk | `PciCore::find_dvsec_capability` |
| `pci_get_dsn()` | per-DSN read | `PciCore::get_dsn` |
| `pci_find_parent_resource()` | per-BAR parent-window lookup | `PciCore::find_parent_resource` |
| `pci_find_resource()` | per-BAR-contains lookup | `PciCore::find_resource` |
| `pci_select_bars()` | per-flag BAR mask | `PciCore::select_bars` |
| `pci_request_region()` / `_regions()` / `_selected_regions()` (+ `_exclusive`) | per-BAR reservation | `PciCore::request_region*` |
| `pci_release_region()` / `_regions()` / `_selected_regions()` | per-BAR release | `PciCore::release_region*` |
| `pci_ioremap_bar()` / `pci_ioremap_wc_bar()` | per-BAR ioremap | `PciCore::ioremap_bar` / `_wc` |
| `pci_remap_iospace()` / `pci_unmap_iospace()` | per-host-bridge IO-space window | `PciCore::remap_iospace` |
| `pci_register_io_range()` / `pci_pio_to_address()` / `pci_address_to_pio()` | per-domain IO-PIO map | `PciCore::register_io_range` |
| `pci_ea_init()` | per-EA-cap parse | `PciCore::ea_init` |
| `pci_enable_device()` / `_mem()` / `_flags()` | per-enable | `PciCore::enable_device*` |
| `pci_reenable_device()` | per-resume re-enable | `PciCore::reenable_device` |
| `pci_disable_device()` / `_enabled_device()` | per-disable | `PciCore::disable_device*` |
| `pci_set_master()` / `pci_clear_master()` | per-bus-master | `PciCore::set_master` |
| `pci_set_mwi()` / `_try_` / `_clear_` / `pci_set_cacheline_size()` | per-MWI | `PciCore::set_mwi` |
| `pci_intx()` | per-INTx mask/unmask | `PciCore::intx` |
| `pci_set_power_state()` / `_locked()` / `pci_power_up()` | per-PM transition | `PciCore::set_power_state*` |
| `pci_save_state()` / `pci_restore_state()` | per-D3-save/restore | `PciCore::save_state` / `restore_state` |
| `pci_store_saved_state()` / `pci_load_saved_state()` / `_and_free_` | per-vfio save | `PciCore::store_saved_state` |
| `pci_add_cap_save_buffer()` / `_ext_` | per-cap save-buffer | `PciCore::add_cap_save_buffer` |
| `pci_allocate_cap_save_buffers()` / `_free_` | per-bring-up | `PciCore::allocate_cap_save_buffers` |
| `pci_pm_init()` | per-device PM cap parse | `PciCore::pm_init` |
| `pci_pme_active()` / `pci_pme_capable()` / `_restore()` / `_wakeup()` / `_wakeup_bus()` | per-PME | `PciCore::pme_*` |
| `pci_enable_wake()` / `pci_wake_from_d3()` | per-wake-enable | `PciCore::enable_wake` |
| `pci_dev_run_wake()` / `pci_dev_need_resume()` | per-runtime-PM query | `PciCore::dev_run_wake` |
| `pci_choose_state()` | per-suspend-state choose | `PciCore::choose_state` |
| `pci_prepare_to_sleep()` / `pci_back_from_sleep()` | per-S-state path | `PciCore::prepare_to_sleep` |
| `pci_finish_runtime_suspend()` | per-runtime-PM final | `PciCore::finish_runtime_suspend` |
| `pci_bridge_d3_possible()` / `_update()` | per-bridge D3 policy | `PciCore::bridge_d3_*` |
| `pci_d3cold_enable()` / `_disable()` | per-device D3cold gate | `PciCore::d3cold_enable` |
| `pci_acs_init()` / `pci_enable_acs()` / `pci_std_enable_acs()` / `__pci_config_acs()` | per-ACS-cap config | `PciCore::acs_*` |
| `pci_acs_enabled()` / `_path_enabled()` / `_flags_enabled()` | per-IOMMU group calc | `PciCore::acs_enabled` |
| `pci_request_acs()` | per-IOMMU-driver vote | `PciCore::request_acs` |
| `pci_configure_ari()` | per-bridge ARI fwd-enable | `PciCore::configure_ari` |
| `pci_enable_atomic_ops_to_root()` | per-AtomicOps-egress negotiate | `PciCore::enable_atomic_ops_to_root` |
| `pcie_get_mps()` / `pcie_set_mps()` | per-Max-Payload-Size | `PciCore::pcie_mps` |
| `pcie_get_readrq()` / `pcie_set_readrq()` | per-Max-Read-Req | `PciCore::pcie_readrq` |
| `pcie_link_speed_mbps()` / `pcie_bandwidth_available()` / `_capable()` | per-link query | `PciCore::pcie_link_*` |
| `pcie_get_supported_speeds()` / `_speed_cap()` / `_width_cap()` | per-link-cap | `PciCore::pcie_link_caps` |
| `pcie_wait_for_link()` / `_delay()` / `_link_status()` | per-link-up poll | `PciCore::pcie_wait_for_link` |
| `pcie_retrain_link()` | per-Retrain-Link bit | `PciCore::pcie_retrain_link` |
| `pcie_print_link_status()` / `__pcie_print_link_status()` | per-boot banner | `PciCore::pcie_print_link_status` |
| `pcie_flr()` / `pcie_reset_flr()` / `pci_af_flr()` / `pci_pm_reset()` | per-reset method | `PciCore::reset_*` |
| `pci_init_reset_methods()` | per-bring-up reset-method probe | `PciCore::init_reset_methods` |
| `pci_reset_function()` / `_locked()` / `_try_` / `___locked()` | per-FN reset | `PciCore::reset_function*` |
| `pci_reset_bus()` / `_try_` / `_probe_` / `pci_bus_error_reset()` | per-bus reset | `PciCore::reset_bus*` |
| `pci_bridge_secondary_bus_reset()` / `pci_reset_secondary_bus()` / `pci_bridge_wait_for_secondary_bus()` | per-2nd-bus reset | `PciCore::bridge_secondary_bus_reset` |
| `pci_dev_lock()` / `_trylock()` / `_unlock()` | per-device-mutex | `PciCore::dev_lock` |
| `pci_dev_save_and_disable()` / `pci_dev_restore()` | per-reset save+restore | `PciCore::dev_save_and_disable` |
| `pci_add_dma_alias()` / `pci_devs_are_dma_aliases()` | per-DMA-alias | `PciCore::add_dma_alias` |
| `pci_device_is_present()` | per-config-read presence | `PciCore::device_is_present` |
| `pci_ignore_hotplug()` | per-suppress-removal-event | `PciCore::ignore_hotplug` |
| `pci_bus_find_domain_nr()` / `_release_` / `_find_emul_` / `_release_emul_` | per-domain alloc | `PciCore::domain_nr_*` |
| `pci_set_vga_state()` | per-VGA-arb plumb | `PciCore::set_vga_state` |
| `pci_reassigndev_resource_alignment()` / `_specified_` / `_request_` | per-VFIO realign | `PciCore::reassign_alignment` |
| `pci_setup()` / `pci_realloc_setup_params()` | per-cmdline parse | `PciCore::cmdline_setup` |

Cross-file callouts (cited but defined elsewhere, declared in `drivers/pci/pci.h`):

| Upstream symbol | Defined in | Role here |
|---|---|---|
| `pci_save_aer_state()` / `pci_restore_aer_state()` / `pci_aer_clear_status()` | `pcie/aer.c` | called by `pci_save_state` / `_restore_state` |
| `pci_save_dpc_state()` / `_restore_` | `pcie/dpc.c` | called by save/restore |
| `pci_save_ptm_state()` / `_restore_` | `pcie/ptm.c` | called by save/restore |
| `pci_save_tph_state()` / `_restore_` | `tph.c` | called by save/restore |
| `pci_save_vc_state()` / `_restore_` | `vc.c` | called by save/restore |
| `pci_restore_pasid_state()` / `_pri_` / `_ats_` / `_iov_` / `_rebar_` | `ats.c` / `iov.c` / `setup-bus.c` | called by `_restore_state` |
| `pcibios_enable_device()` / `pcibios_set_master()` / `pcibios_disable_device()` / `pcibios_device_add()` / `pcibios_release_device()` / `pcibios_reset_secondary_bus()` / `pcibios_set_pcie_reset_state()` | arch `arch/<arch>/pci/*` | weak symbols overridden per-arch |
| `pcie_aspm_powersave_config_link()` | `pcie/aspm.c` | called from `do_pci_enable_device` |
| `pci_msi_init()` / `pci_msix_init()` / `pci_restore_msi_state()` | `msi/` | called from `pci_pm_init` / `pci_restore_state` |
| `pci_create_root_bus()` / `pci_scan_root_bus()` / `pci_scan_root_bus_bridge()` | `probe.c` | enumeration entry — pairs with `pci_enable_device` consumer |
| `__pci_register_driver()` / `pci_unregister_driver()` | `pci-driver.c` | driver-side bind — pairs with `pci_enable_device` |
| `pci_dev_specific_enable_acs()` / `_disable_acs_redir()` / `pci_fixup_device()` | `quirks.c` | called from `pci_enable_acs` / `do_pci_enable_device` |

## Compatibility contract

REQ-1: Capability walk (`pci_find_capability(dev, cap)`):
- pos = `__pci_bus_find_cap_start(dev.bus, dev.devfn, dev.hdr_type)`.
- `__pci_bus_find_cap_start`: read `PCI_STATUS`; if `PCI_STATUS_CAP_LIST` clear → 0. Else hdr_type switch: NORMAL/BRIDGE → `PCI_CAPABILITY_LIST` (= 0x34); CARDBUS → `PCI_CB_CAPABILITY_LIST` (= 0x14); else 0.
- pos = `__pci_find_next_cap(dev.bus, dev.devfn, pos, cap)`: walk cap-id list at `pos`, follow `PCI_CAP_LIST_NEXT` bytes (mask off low 2 bits per spec), bounded by `PCI_FIND_NEXT_CAP_TTL` (= 48) to bound malformed loops.
- Returns first-match offset ∈ [0x40, 0xFF] in config space, or 0.
- `pci_find_next_capability(dev, pos, cap)` resumes from `pos + PCI_CAP_LIST_NEXT` (= +1).
- `pci_bus_find_capability(bus, devfn, cap)` variant for pre-`pci_dev` enumeration (called from `probe.c::pci_scan_device`).

REQ-2: Extended-capability walk (`pci_find_ext_capability(dev, cap)`):
- If `dev.cfg_size <= PCI_CFG_SPACE_SIZE` (= 256B): return 0 (legacy PCI cfg, no PCIe ext space).
- Start at `PCI_CFG_SPACE_SIZE` (= 0x100). Each ext-cap header is `u32`: bits[15:0]=cap-id, [19:16]=ver, [31:20]=next-pos.
- TTL-bounded walk via `PCI_FIND_NEXT_EXT_CAP` macro. Returns offset ∈ [0x100, 0xFFF], or 0.
- `pci_find_next_ext_capability(dev, start, cap)` allows resume (VSEC/DVSEC may repeat).
- `pci_get_dsn(dev)` reads `PCI_EXT_CAP_ID_DSN` (= 0x03) + 4 → two `dword`s → low-half + high-half → returns 64-bit DSN.
- `pci_find_vsec_capability(dev, vendor, cap)` filters `PCI_EXT_CAP_ID_VNDR` (= 0x0B) by VSEC-vendor-id in `PCI_VNDR_HEADER`.
- `pci_find_dvsec_capability(dev, vendor, dvsec)` filters `PCI_EXT_CAP_ID_DVSEC` (= 0x23) by `PCI_DVSEC_HEADER1.vendor` + `PCI_DVSEC_HEADER2.id`.

REQ-3: BAR resource access:
- `pci_find_parent_resource(dev, res)`: iterate `dev.bus.resource[]` (host-bridge windows); return first `r` with `resource_contains(r, res)` ∧ (prefetch mismatch → return NULL). Caller is `pci_assign_resource`.
- `pci_find_resource(dev, res)`: scan `dev.resource[0..PCI_STD_NUM_BARS-1]` (= 6), return first containing `res`. Caller is `pci_mmap_resource`.
- `pci_resource_name(dev, i)`: returns "BAR 0..5" / "ROM" / "VF BAR 0..5" / "bridge window" string for sysfs.
- `pci_select_bars(dev, flags)`: returns bitmask of `i` where `dev.resource[i].flags & flags` nonzero (for `IORESOURCE_MEM` ∨ `IORESOURCE_IO` filtering).
- `pci_request_region(dev, bar, name)` → `__pci_request_region(dev, bar, name, 0)` → `__request_region(parent, start, end-start+1, name, 0)` registers the BAR window with the resource tree (preventing dual claim).
- `pci_request_regions(dev, name)`: loops bars 0..5 + ROM; rollback on failure. `pci_request_selected_regions(dev, bars, name)`: per `bars` mask. `_exclusive` variant passes `IORESOURCE_EXCLUSIVE`.
- `pci_release_region(dev, bar)`: `release_resource(&dev.resource[bar])`.
- `pci_ioremap_bar(dev, bar)` / `pci_ioremap_wc_bar(dev, bar)`: validates `IORESOURCE_MEM` + non-zero start; `ioremap` or `ioremap_wc` the BAR.

REQ-4: PIO + IO-space remapping:
- `pci_remap_iospace(res, phys_addr)`: validates `IORESOURCE_IO`, `res.start <= res.end`, `res.end < IO_SPACE_LIMIT`. Maps `PCI_IOBASE + res.start` → `phys_addr` PAGE-by-PAGE via `ioremap_page_range` with `pgprot_device` (the host-bridge's IO window is a CPU-side virtual region the `inX`/`outX` insn references).
- `pci_unmap_iospace(res)`: `vunmap_range(...)`.
- `pci_register_io_range(fwnode, addr, size)` records the mapping in a `logical_pio_list` so `pci_address_to_pio(phys)` / `pci_pio_to_address(pio)` translate between the host-bridge IO MMIO aperture and the synthetic IO port ABI.

REQ-5: `pci_enable_device(dev)` (= `pci_enable_device_flags(dev, IORESOURCE_MEM | IORESOURCE_IO)`):
- `pci_update_current_state(dev, dev.current_state)`: refresh `PCI_PM_CTRL.power_state`.
- If `atomic_inc_return(&dev.enable_cnt) > 1`: return 0 (already enabled — refcount-style).
- Else: `bridge = pci_upstream_bridge(dev)`; if `bridge`: `pci_enable_bridge(bridge)` (recursive: bridge must be in D0 + bus-master + PCI_COMMAND_MEM set).
- Build `bars` mask: scan `resource[0..PCI_ROM_RESOURCE]` + `resource[PCI_BRIDGE_RESOURCES..DEVICE_COUNT_RESOURCE]` (skip SRIOV-VF BARs) for `flags & req_flags`.
- `do_pci_enable_device(dev, bars)`:
  - `pci_set_power_state(dev, PCI_D0)`. If `err < 0 ∧ err != -EIO`: error.
  - `bridge = pci_upstream_bridge`; if `bridge`: `pcie_aspm_powersave_config_link(bridge)`.
  - `pci_host_bridge_enable_device(dev)` (CXL: claim coherent-DMA).
  - `pcibios_enable_device(dev, bars)` (arch hook — x86: `pci_enable_resources` writes `PCI_COMMAND.MEM|IO` and warns if BAR not aligned to window).
  - `pci_fixup_device(pci_fixup_enable, dev)` (quirks).
  - If `!dev.msi_enabled ∧ !dev.msix_enabled`: read `PCI_INTERRUPT_PIN`; if nonzero ∧ `PCI_COMMAND.INTX_DISABLE` set: clear `INTX_DISABLE` (legacy INTx routing demands the bit clear).
- On error: `atomic_dec(&dev.enable_cnt)`.

REQ-6: `pci_disable_device(dev)`:
- `if (atomic_sub_return(1, &dev.enable_cnt) != 0): return` (refcount).
- `pci_dev_assert_locked(dev)` (caller must hold pci_dev mutex if SRIOV).
- `do_pci_disable_device(dev)`: read `PCI_COMMAND`; if `MASTER` set: clear it (stop bus-master DMA). `pcibios_disable_device(dev)`.
- Clear `dev.is_busmaster = 0`.

REQ-7: `pci_set_master(dev)`:
- `__pci_set_master(dev, true)`: read `PCI_COMMAND`, OR-in `PCI_COMMAND_MASTER`, write-back, set `dev.is_busmaster = true`.
- `pcibios_set_master(dev)` (arch hook — pre-PCIe: clamp `PCI_LATENCY_TIMER` to `pcibios_max_latency` (default 255); PCIe: skip — latency timer reserved).

REQ-8: PM-state transitions:
- `pci_set_power_state(dev, state)` → `__pci_set_power_state(dev, state, locked=false)`.
- `pci_power_up(dev)`: `__pci_set_power_state(dev, PCI_D0, locked=false)`.
- Transitions to / from PCI_D0:
  - To D0: `pci_set_full_power_state(dev, locked)`: `pci_platform_power_transition(dev, PCI_D0)` → ACPI `_PS0`; clear PME-status; refresh `dev.current_state`.
  - To D1/D2/D3hot: `pci_set_low_power_state(dev, state, locked)`: write `PCI_PM_CTRL.power_state`; `msleep(pci_pm_d3hot_delay)` if D3hot; `pci_platform_power_transition(dev, state)`.
  - To D3cold: requires `pci_bridge_d3_possible(bridge)` ∧ `dev.d3cold_allowed`. Bridge powers off via ACPI `_PS3` / `_OFF`.
- `pci_choose_state(dev, pm_message_t state)`: maps suspend `pm_message_t` → `pci_power_t` (target).
- `pci_update_current_state(dev, state)`: read `PCI_PM_CTRL`, set `dev.current_state` to the actual state.
- `pci_refresh_power_state(dev)`: `pci_platform_pm_init` + `pci_update_current_state`. Called after suspend-resume of host bridge.

REQ-9: PME (Power-Management Event):
- `pci_pme_capable(dev, state)`: returns `dev.pme_support & (1 << state)`.
- `pci_pme_active(dev, enable)`: read `PCI_PM_CTRL`, set/clear `PCI_PM_CTRL_PME_ENABLE` (= 0x0100). If `dev.pme_poll`: add/remove from `pci_pme_list` (worker `pci_pme_list_scan` runs every `PME_TIMEOUT` (= 1000ms) — polls `PCI_PM_CTRL.PME_STATUS` for hardware that doesn't generate PME irq).
- `pci_pme_wakeup(dev, pme_poll_reset)`: pme-status check; if pending: `pci_check_pme_status` + `pm_request_resume`. Reset pme-status. Returns `0`.
- `pci_pme_wakeup_bus(bus)`: walk bus → `pci_pme_wakeup` each dev.
- `pci_pme_restore(dev)`: after resume, re-write PME_ENABLE.

REQ-10: Wake-from-S/Dx:
- `pci_enable_wake(dev, state, enable)` → `__pci_enable_wake(dev, state, enable)`: `device_set_wakeup_enable(dev.dev, enable)` (driver-core wakeup flag); `__pci_pme_active(dev, enable)`; `__pci_enable_wake_platform(dev, enable)` (ACPI `_DSW`).
- `pci_wake_from_d3(dev, enable)`: `pci_enable_wake(dev, PCI_D3hot, enable)` ∧ `pci_enable_wake(dev, PCI_D3cold, enable)`.
- `pci_dev_run_wake(dev)`: returns true if dev has runtime-wake (PME-capable + parent runtime-PM cap + ACPI wakeup-enable).
- `pci_dev_need_resume(dev)`: returns true if dev needs explicit resume on system-resume.

REQ-11: D3cold policy:
- `pci_bridge_d3_possible(bridge)`: returns true iff `pci_is_pcie(bridge)` ∧ type ∈ {ROOT_PORT, DOWNSTREAM} ∧ not in `bridge_d3_blacklist` DMI quirks ∧ `pci_bridge_d3_force` ∨ (`acpi_pci_bridge_d3(bridge)` ∧ slot supports L1 substate).
- `pci_bridge_d3_update(dev)`: when a child gains/loses D3cold support, walk parent chain and recompute bridge `bridge_d3` flag.
- `pci_d3cold_enable(dev)` / `_disable(dev)`: toggle `dev.no_d3cold` flag (no_d3cold==true blocks D3cold).
- `pci_dev_check_d3cold(dev, data)`: callback per-child; if any child says no, parent cannot d3cold.

REQ-12: Saved-state (`pci_save_state(dev)`):
- For i = 0..15: `pci_read_config_dword(dev, i*4, &dev.saved_config_space[i])` (64-byte header).
- Cap-buffers (per-cap allocated by `pci_allocate_cap_save_buffers`): `pci_save_pcie_state` (DEVCTL/LNKCTL/SLTCTL/RTCTL/DEVCTL2/LNKCTL2/SLTCTL2), `pci_save_pcix_state`, `pci_save_dpc_state`, `pci_save_aer_state`, `pci_save_ptm_state`, `pci_save_tph_state`, `pci_save_vc_state`.
- Set `dev.state_saved = true`.

REQ-13: `pci_restore_state(dev)` (per-order):
- `pci_restore_pcie_state` → PASID → PRI → ATS → VC → ReBAR → DPC → PTM → TPH (extended caps first since they may gate later behavior).
- `pci_aer_clear_status` + `pci_restore_aer_state` (clear before restore).
- `pci_restore_config_space`: per-hdr_type. NORMAL: restore words [10..15] (cap+IRQ) first, then [4..9] (BARs, retry=10 since BAR writes may take cycles), then [0..3] (vendor/dev/cmd — last so CMD is final write). BRIDGE: [12..15], then [9..11] (with `force=true` on prefetch regs — Intel S3 quirk), then [0..8]. CARDBUS: [0..15] in order.
- `pci_restore_pcix_state` + `pci_restore_msi_state` + `pci_enable_acs` + `pci_restore_iov_state`.
- Clear `dev.state_saved = false`.

REQ-14: `pci_store_saved_state(dev)`: allocates `struct pci_saved_state` (flat `u32 config_space[16]` + `struct pci_cap_saved_data cap[]` flexible array), copies saved state; used by `vfio-pci` for migration. `pci_load_saved_state(dev, state)` restores in-memory copy (no hw write). `pci_load_and_free_saved_state` = load + free.

REQ-15: Cap save-buffers:
- `pci_add_cap_save_buffer(dev, cap, size)`: alloc `struct pci_cap_saved_state` of `size`, add to `dev.saved_cap_space` linked list keyed by cap-id.
- `pci_add_ext_cap_save_buffer(dev, cap, size)`: same, but `cap` is ext-cap-id and `extended` flag set.
- `pci_allocate_cap_save_buffers(dev)`: called from `pci_pm_init`; pre-allocates buffers for PCIE (28B), PCIX (4B), HT-MSI mapping, AER, IOV, DPC, PTM, TPH, ASPM-L1ss, VC.
- `pci_free_cap_save_buffers(dev)`: free-on-`pci_destroy_dev`.
- `pci_find_saved_cap(dev, cap)` / `_ext_cap`: linked-list lookup.

REQ-16: Enhanced Allocation (EA):
- `pci_ea_init(dev)`: locate `PCI_CAP_ID_EA` (= 0x14); read header (entry-count for hdr_type=NORMAL is bits[21:16], for BRIDGE bits[19:16] + fixed-sub-bridge-info); for each entry call `pci_ea_read(dev, offset)`.
- `pci_ea_read`: parse first-dword entry header (entry-size, BAR-equiv-indicator BEI, primary/secondary base + max-offset, properties P/SP, writable, enabled, prefetchable, mem/io). Compute resource start/end, flags via `pci_ea_flags`. Store into `dev.resource[bei]` if EA enabled (EA overrides legacy BAR sizing).
- `pci_ea_get_resource(dev, bei, prop)`: pick `dev.resource[]` slot for BEI value.

REQ-17: ACS (Access Control Services):
- `pci_request_acs()`: set static `pci_acs_enable = 1` (called by IOMMU drivers at init — IOMMU groups need ACS to ensure peer-isolation).
- `pci_acs_init(dev)`: cache `dev.acs_cap = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_ACS)`; read `dev.acs_capabilities` (= `PCI_ACS_CAP`).
- `pci_enable_acs(dev)`: if `pci_acs_enable` ∧ `pci_dev_specific_enable_acs(dev)` returns true (quirks): `pci_std_enable_acs(dev, &caps)` (SV | RR | CR | UF, plus TB for external-facing/untrusted/ats-disabled); then `__pci_config_acs(dev, &caps, disable_acs_redir_param, ...)` + `__pci_config_acs(dev, &caps, config_acs_param, ...)` apply command-line overrides. Write `caps.ctrl` → `PCI_ACS_CTRL`.
- `pci_acs_enabled(dev, acs_flags)`: returns true iff requested ACS flags are set in `dev`'s ACS-CTRL. For non-PCIe parents, defer to `pci_dev_specific_acs_enabled` quirks. For Multi-Function devices, mandate `SV|RR|CR|UF`. For root-complex integrated endpoints, return true (no peer to redirect).
- `pci_acs_path_enabled(start, end, acs_flags)`: walk from `start` up to `end` (or root if `end==NULL`); all bridges on path must have `acs_flags` enabled. Used by IOMMU-group computation.

REQ-18: ARI + AtomicOps:
- `pci_configure_ari(dev)`: if `dev` is multi-function PCIe ∧ upstream bridge has `PCI_EXP_DEVCAP2.ARI_FWD`: set `PCI_EXP_DEVCTL2.ARI` on bridge (enables 8-bit function-number space → up to 256 functions per device, used by SRIOV).
- `pci_enable_atomic_ops_to_root(dev, cap_mask)`: walk every upstream port (RC/switch/root-port); each must have `PCI_EXP_DEVCAP2.ATOMIC_OP_ROUTING_SUPP` ∧ `dev` egress must be enabled via `PCI_EXP_DEVCTL2.ATOMIC_OP_REQ_EN`. cap_mask = `PCI_EXP_DEVCAP2_ATOMIC_COMP32|64|128`. Used by NVIDIA GPU + some NICs.

REQ-19: PCIe link helpers:
- `pcie_get_mps(dev)`: read `PCI_EXP_DEVCTL.MPS`, return 128 << field.
- `pcie_set_mps(dev, mps)`: clamp mps ≤ `dev.pcie_mpss`, write `PCI_EXP_DEVCTL.MPS`.
- `pcie_get_readrq(dev)`: read `PCI_EXP_DEVCTL.READRQ`, return 128 << field.
- `pcie_set_readrq(dev, rq)`: validate against host-bridge clamp (`pcie_bus_config`).
- `pcie_link_speed_mbps(pdev)`: read `PCI_EXP_LNKSTA.CLS`, return MB/s.
- `pcie_bandwidth_available(dev, limiting_dev, speed, width)`: walk upstream, compute minimum link bandwidth + which device limits.
- `pcie_get_supported_speeds(dev)`: returns bitmask of supported speeds (LNKCAP2 if present).
- `pcie_wait_for_link(pdev, active)`: poll `PCI_EXP_LNKSTA.DLLLA` (Data-Link Layer Link Active) for transition; timeout `PCIE_LINK_RETRAIN_TIMEOUT_MS`.
- `pcie_retrain_link(pdev, use_lt)`: set `PCI_EXP_LNKCTL.LR` (Retrain Link); poll `LNKSTA.LT` or `LNKSTA.DLLLA` per `use_lt`.

REQ-20: Reset methods (`dev.reset_methods[]` ordered list):
- `pci_init_reset_methods(dev)`: probe each reset method, fill `reset_methods[0..PCI_NUM_RESET_METHODS-1]` (FLR (PCIe) → AF-FLR (legacy PCI Advanced-Features cap) → PM (PCI_PM_CTRL D3hot→D0 reset) → bus (parent-bus-reset) → cxl-bus (CXL-only soft-reset). Call each method with `probe=true` to check support without performing the reset.
- `pcie_flr(dev)`: read `PCI_EXP_DEVCAP.FLR_CAP`; set `PCI_EXP_DEVCTL.BCR_FLR`; wait `pci_dev_wait(dev, "FLR", PCIE_RESET_READY_POLL_MS=60000ms)`.
- `pci_af_flr(dev, probe)`: cap `PCI_CAP_ID_AF` + `PCI_AF_CAP.FLR`; write `PCI_AF_CTRL.FLR`.
- `pci_pm_reset(dev, probe)`: pre-PCIe FN reset via D3hot transition with reset bit.
- `pci_reset_function(dev)`: lock + save + iterate `reset_methods[]` → first matching → save+restore. `_locked` skips locking. `pci_try_reset_function` returns -EAGAIN if can't lock.

REQ-21: Bus / slot reset:
- `pci_bridge_secondary_bus_reset(bridge)`: set `PCI_BRIDGE_CTL.BUS_RESET` in bridge `PCI_PRIMARY_BUS+2`; sleep `PCI_PM_D3HOT_WAIT` (= 10ms); clear; `pci_bridge_wait_for_secondary_bus(bridge, "bus-reset")`.
- `pci_bus_reset(bus, probe)`: lock all dev on bus, save+disable all, secondary-bus-reset on bus's bridge, restore all.
- `pci_slot_reset(slot, probe)`: same for slot scope.
- `pci_reset_bus(pdev)`: from device → walk to bridge → `pci_bus_reset(bus)`.
- `pci_bridge_wait_for_secondary_bus(bridge, reset_type)`: poll for downstream link-up; per device `d3cold_delay` (default `PCI_PM_D3COLD_WAIT` = 100ms). Up to `PCIE_RESET_READY_POLL_MS` = 60000ms.

REQ-22: Dev locks:
- `pci_dev_lock(dev)`: acquire `dev.dev.mutex` + `dev_pm_lock`. Used pre-reset.
- `pci_dev_trylock(dev)`: best-effort.
- `pci_bus_lock(bus)` / `_unlock` / `_trylock`: walk bus + lock every dev (in-order); for cascaded brides recurses.
- `pci_slot_lock(slot)` / `_unlock` / `_trylock`: per-slot walk.

REQ-23: DMA-alias + presence:
- `pci_add_dma_alias(dev, devfn_from, nr_devfns)`: register alias mapping (e.g., legacy PCI-to-PCI bridge requesters appear as bridge's `bus:0:0`). IOMMU group uses this.
- `pci_devs_are_dma_aliases(dev1, dev2)`: returns true if dev1 maps to dev2 or vice versa.
- `pci_device_is_present(pdev)`: `pci_bus_read_config_word(bus, devfn, PCI_VENDOR_ID)` ≠ 0xFFFF (surprise-removal detection).
- `pci_ignore_hotplug(dev)`: set `dev.ignore_hotplug = 1` (used by NVMe + drivers that handle their own remove).

REQ-24: Domain numbering:
- `pci_bus_find_domain_nr(bus, parent)`: per-arch (x86: returns OF/ACPI domain or via `pci_bus_find_emul_domain_nr` for guest emul).
- `pci_bus_release_domain_nr(parent, dn)`: per-arch.
- `pci_bus_find_emul_domain_nr(hint, min, max)`: IDA-allocates a domain-nr ∈ [`min`, `max`] for emulated host bridges (hyperv-pci, etc.). `_release_` returns to IDA.

REQ-25: VGA arb:
- `pci_set_vga_state(dev, decode, command_bits, flags)`: enable/disable VGA decoding for legacy IO ports (0x3B0-0x3DF) + memory range (0xA0000-0xBFFFF) on dev; flag `PCI_VGA_STATE_CHANGE_BRIDGE` propagates VGA Enable bit on every upstream PCI-to-PCI bridge.
- `pci_register_set_vga_state(func)`: arch registers `arch_set_vga_state_t` callback (x86 → `vga_set_legacy_decoding`).

REQ-26: Resource-alignment overrides:
- `resource_alignment_param` (sysfs `/sys/bus/pci/resource_alignment`, also `pci=resource_alignment=...` cmdline): per-device `<order>[@<bus:dev.func>]` rules forcing a BAR to a larger alignment so VFIO can mmap a full PAGE without other devices crowding it.
- `pci_reassigndev_resource_alignment(dev)`: per-device on `pci_assign_resource` path; sets `IORESOURCE_STARTALIGN` and bumps size accordingly.
- `pci_request_resource_alignment(dev, bar, align, resize)`: helper.

REQ-27: Command-line parser (`pci_setup(str)` early_param):
- Tokens: `off`, `bios`, `nobios`, `nomsi`, `noaer`, `nodomains`, `noari`, `noats`, `noacs`, `cbiosize=N`, `cbmemsize=N`, `realloc=on|off`, `assign-busses`, `hpiosize=N`, `hpmmiosize=N`, `hpmmioprefsize=N`, `hpbussize=N`, `pcie_bus_safe`, `pcie_bus_perf`, `pcie_bus_peer2peer`, `pcie_bus_tune_off`, `pcie_port_pm=off|force`, `early_dump`, `disable_acs_redir=...`, `config_acs=...`, `resource_alignment=...`.
- `pci_realloc_setup_params()` (subsys_initcall): apply parsed alignment params.

REQ-28: PCI-X mode (legacy):
- `pcix_get_mmrbc(dev)` / `_max_mmrbc()` / `pcix_set_mmrbc(dev, mmrbc)`: Max Memory Read Byte Count (512 / 1024 / 2048 / 4096).

## Acceptance Criteria

- [ ] AC-1: `pci_find_capability(dev, PCI_CAP_ID_EXP)` returns valid offset for any PCIe-capable device (cap_list bit set + walk terminates ≤ 48 iters).
- [ ] AC-2: `pci_find_ext_capability(dev, PCI_EXT_CAP_ID_ERR)` returns 0 on PCI-not-PCIe devices (`cfg_size <= 256`) and non-zero AER-cap offset on PCIe devices supporting AER.
- [ ] AC-3: `pci_enable_device(dev)` on already-enabled device: returns 0; `enable_cnt` increments; idempotent.
- [ ] AC-4: `pci_enable_device(dev)` on disabled device: ASPM configured, `PCI_COMMAND.MEM|IO` set, INTx-disable cleared if pin nonzero, ref-counts symmetric with `pci_disable_device`.
- [ ] AC-5: `pci_set_master(dev)` sets `PCI_COMMAND.BUS_MASTER`; on PCIe, leaves `PCI_LATENCY_TIMER` untouched; on PCI, clamps to `pcibios_max_latency`.
- [ ] AC-6: `pci_save_state(dev)` followed by `pci_restore_state(dev)` round-trips first 64 bytes of config space + every saved-cap buffer; state_saved flag toggles true → false.
- [ ] AC-7: D3hot save/restore: `pci_set_power_state(dev, PCI_D3hot)` followed by `pci_set_power_state(dev, PCI_D0)` + `pci_restore_state` produces identical config + MSI vector state.
- [ ] AC-8: `pcie_flr(dev)`: device with `PCI_EXP_DEVCAP.FLR_CAP` sees `BCR_FLR` bit asserted; `pci_dev_wait` polls `PCI_VENDOR_ID` until ≠ 0xFFFF up to 60s.
- [ ] AC-9: `pci_reset_function(dev)`: with FLR-capable device, calls `pcie_flr` after save_and_disable + completes with restore.
- [ ] AC-10: `pci_acs_path_enabled(start, end, PCI_ACS_RR|PCI_ACS_CR|PCI_ACS_SV|PCI_ACS_UF)`: returns false if any upstream bridge has ACS-disabled or missing-ACS-cap.
- [ ] AC-11: `pci_enable_atomic_ops_to_root(dev, PCI_EXP_DEVCAP2_ATOMIC_COMP64)` returns 0 only when every upstream switch + RC supports 64-bit AtomicOps routing.
- [ ] AC-12: `pci_request_region(dev, 0, "drv")` followed by another `pci_request_region(dev, 0, ...)` from same caller: second returns -EBUSY; release-then-rerequest succeeds.
- [ ] AC-13: `pci_get_dsn(dev)` returns 64-bit DSN exactly matching `lspci -vvv | grep "Device Serial Number"` byte order.
- [ ] AC-14: `pci_bridge_d3_possible(bridge)` returns false on DMI-blacklisted bridges (e.g., specific Apple platforms) even when PCIe + root-port.
- [ ] AC-15: `pci_setup("nomsi")` cmdline: sets `pci_no_msi`; `pci_msi_enable` returns -ENOSYS thereafter.
- [ ] AC-16: `pci_dev_run_wake(dev)` returns true only when (PME-capable + parent runtime-PM + ACPI wakeup enabled).
- [ ] AC-17: `pci_disable_device(dev)`: clears `PCI_COMMAND.BUS_MASTER` (stops DMA before driver unload).
- [ ] AC-18: SRIOV-VF disable + re-enable via VF driver: `pci_restore_iov_state` re-applies VF NumVFs through `pci_restore_state`.

## Architecture

```
struct PciDev {
  bus: *PciBus,
  devfn: u32,                  // bus:dev.func encoded
  vendor: u16,
  device: u16,
  subsystem_vendor: u16,
  subsystem_device: u16,
  class: u32,
  revision: u8,
  hdr_type: u8,                // NORMAL | BRIDGE | CARDBUS
  pcie_cap: u8,                // offset of PCI_CAP_ID_EXP, 0 = not PCIe
  pcie_mpss: u8,               // PCI_EXP_DEVCAP.MPSS
  pcie_flr_quirk: bool,
  cfg_size: u32,               // 256 (PCI) or 4096 (PCIe)
  acs_cap: u16,                // ext-cap offset of ACS, 0 = not present
  acs_capabilities: u16,       // PCI_ACS_CAP cached
  saved_config_space: [u32; 16],
  saved_cap_space: List<PciCapSavedState>,
  state_saved: bool,
  enable_cnt: AtomicI32,       // refcounted pci_enable_device
  is_busmaster: bool,
  msi_enabled: bool,
  msix_enabled: bool,
  current_state: u8,           // PCI_D0..D3cold
  pm_cap: u8,                  // offset of PCI_CAP_ID_PM, 0 = not present
  pme_support: u8,             // bitmask: bit n = wake-from-Dn
  pme_poll: bool,              // pci_pme_list polling
  d3hot_delay: u32,            // override pci_pm_d3hot_delay
  d3cold_delay: u32,
  d3cold_allowed: bool,
  no_d3cold: bool,
  bridge_d3: bool,             // computed by pci_bridge_d3_update
  ignore_hotplug: bool,
  external_facing: bool,
  untrusted: bool,
  resource: [Resource; PCI_NUM_RESOURCES],  // BARs + ROM + bridge-windows + SRIOV-VF-BARs
  reset_methods: [u32; PCI_NUM_RESET_METHODS],
  dma_alias_mask: BitMap<256>, // pci_add_dma_alias
  ...
}

struct PciBus {
  parent: *PciDev,             // bridge dev (NULL for root)
  number: u8,
  primary: u8,
  resource: [*Resource; PCI_BRIDGE_RESOURCE_NUM],
  ...
}

struct PciHostBridge {
  dev: Device,
  windows: ResourceList,       // host-bridge MMIO + IO windows
  dma_ranges: ResourceList,    // OF/ACPI dma-ranges
  ...
}

struct PciAcs { ctrl: u16, fw_ctrl: u16 }
struct PciSavedState { config_space: [u32; 16], cap: [PciCapSavedData] }
struct PciCapSavedState { cap: u16, extended: bool, size: u32, data: [u8] }
struct PciPmeDevice { list: ListNode, dev: *PciDev, work: ... }
```

### `PciCore::find_capability(dev, cap) -> Option<u8>`

1. `status = pci_read_config_word(dev, PCI_STATUS)`.
2. If `status & PCI_STATUS_CAP_LIST == 0`: return `None`.
3. Per-`hdr_type`: pos = NORMAL/BRIDGE → 0x34; CARDBUS → 0x14; else `None`.
4. Loop bounded by TTL = 48:
   - read `[pos]` byte = cap_id; read `[pos+1]` byte = next.
   - if `cap_id == cap`: return `Some(pos)`.
   - if `next == 0`: return `None`.
   - `pos = next & ~0x3` (low 2 bits reserved).
   - if `pos < 0x40`: malformed; return `None`.
5. Return `None`.

### `PciCore::find_ext_capability(dev, cap) -> Option<u16>`

1. If `dev.cfg_size <= 256`: return `None`.
2. `pos = 0x100`.
3. Loop TTL = `(4096 - 0x100) / 8` (= 480):
   - `header = pci_read_config_dword(dev, pos)`.
   - `ext_cap_id = header & 0xFFFF`; `next = (header >> 20) & 0xFFC`.
   - if `ext_cap_id == cap`: return `Some(pos)`.
   - if `next == 0 ∨ next < 0x100`: return `None`.
   - `pos = next`.
4. Return `None`.

### `PciCore::enable_device(dev) -> Result<()>`

1. `pci_update_current_state(dev, dev.current_state)`.
2. If `dev.enable_cnt.fetch_add(1, AcqRel) >= 1`: return `Ok(())`.
3. `bridge = pci_upstream_bridge(dev)`; if `bridge`: `pci_enable_bridge(bridge)` (recurses).
4. Build `bars: u32`: iterate `i ∈ 0..=PCI_ROM_RESOURCE`: if `dev.resource[i].flags & (MEM|IO)`: `bars |= 1<<i`. iterate `i ∈ PCI_BRIDGE_RESOURCES..DEVICE_COUNT_RESOURCE`: same.
5. `do_pci_enable_device(dev, bars)`:
   - `pci_set_power_state(dev, PCI_D0)`; on err≠-EIO return err.
   - `bridge`: `pcie_aspm_powersave_config_link(bridge)`.
   - `pci_host_bridge_enable_device(dev)`.
   - `pcibios_enable_device(dev, bars)` (arch hook).
   - `pci_fixup_device(pci_fixup_enable, dev)`.
   - If !msi_enabled && !msix_enabled: read `PCI_INTERRUPT_PIN`. If pin != 0 ∧ `PCI_COMMAND.INTX_DISABLE` set: clear it.
6. On error: `dev.enable_cnt.fetch_sub(1, AcqRel)`. Return.

### `PciCore::set_master(dev)`

1. `__pci_set_master(dev, enable=true)`:
   - read `PCI_COMMAND` = old.
   - `new = old | PCI_COMMAND_MASTER`.
   - if new != old: write `PCI_COMMAND = new`.
   - `dev.is_busmaster = true`.
2. `pcibios_set_master(dev)` (PCIe: noop; PCI: clamp `PCI_LATENCY_TIMER`).

### `PciCore::save_state(dev) -> Result<()>`

1. For i ∈ 0..16: `dev.saved_config_space[i] = pci_read_config_dword(dev, i*4)`.
2. `dev.state_saved = true`.
3. `pci_save_pcie_state(dev)?`: if PCIe-cap, alloc-on-demand cap-save-buffer of 28 bytes; write `DEVCTL`, `LNKCTL`, `SLTCTL`, `RTCTL`, `DEVCTL2`, `LNKCTL2`, `SLTCTL2`.
4. `pci_save_pcix_state(dev)?`: if PCI-X cap.
5. `pci_save_dpc_state(dev)`, `pci_save_aer_state(dev)`, `pci_save_ptm_state(dev)`, `pci_save_tph_state(dev)`, `pci_save_vc_state(dev)`.
6. Return Ok.

### `PciCore::restore_state(dev)`

1. `pci_restore_pcie_state`, `_pasid_`, `_pri_`, `_ats_`, `_vc_`, `_rebar_`, `_dpc_`, `_ptm_`, `_tph_`.
2. `pci_aer_clear_status`, `pci_restore_aer_state`.
3. `pci_restore_config_space`: per `hdr_type`:
   - NORMAL: ranges (10..15, retry=0), (4..9, retry=10), (0..3, retry=0).
   - BRIDGE: (12..15), (9..11, force=true), (0..8).
   - CARDBUS: (0..15).
4. `pci_restore_pcix_state`, `pci_restore_msi_state`.
5. `pci_enable_acs`, `pci_restore_iov_state`.
6. `dev.state_saved = false`.

### `PciCore::acs_path_enabled(start, end, flags) -> bool`

1. `pdev = start`.
2. Loop:
   - if `!pci_acs_enabled(pdev, flags)`: return false.
   - if `pdev == end`: return true.
   - `pdev = pci_upstream_bridge(pdev)`.
   - if `pdev == None`: return true (reached root).

### `PciCore::reset_function(dev) -> Result<()>`

1. `pci_dev_lock(dev)`.
2. `pci_dev_save_and_disable(dev)`:
   - `pci_save_state(dev)`.
   - `pci_dev_d3_sleep(dev)` (idempotent transition to D0).
   - `pci_clear_master(dev)`.
3. For method in `dev.reset_methods[..]` (zero-terminated):
   - if `method.fn(dev, probe=false)` succeeds: break.
4. `pci_dev_restore(dev)`: `pci_restore_state(dev)`.
5. `pci_dev_unlock(dev)`.

### `PciCore::enable_atomic_ops_to_root(dev, cap_mask) -> Result<()>`

1. If !`pci_is_pcie(dev)`: return Err(-EINVAL).
2. `bridge = pci_upstream_bridge(dev)`.
3. Loop: if `bridge == None`: break.
   - read `PCI_EXP_DEVCAP2(bridge)`.
   - if not all of `cap_mask` supported: return Err(-EINVAL).
   - if pcie-type ∈ {ROOT_PORT, UPSTREAM, DOWNSTREAM}: read `PCI_EXP_DEVCTL2(bridge)`; if `ATOMIC_EGRESS_BLOCK` set: return Err(-EINVAL).
   - if pcie-type == ROOT_PORT: break.
   - `bridge = pci_upstream_bridge(bridge)`.
4. read `PCI_EXP_DEVCTL2(dev)`, set `ATOMIC_OP_REQ_EN`, write-back.
5. Return Ok.

### Resource-name table (per `pci_resource_name`)

| Index | hdr_type=NORMAL/BRIDGE | hdr_type=CARDBUS |
|---|---|---|
| 0..5 | BAR 0..5 | BAR 1, unknown, … |
| 6 (PCI_ROM_RESOURCE) | ROM | unknown |
| 7..12 (PCI_IOV_RESOURCES, if CONFIG_PCI_IOV) | VF BAR 0..5 | unknown |
| PCI_BRIDGE_RESOURCES..+3 | bridge window (io, mem, mem pref) | CardBus bridge window 0/1, mem 0/1 |

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_walk_ttl_bounded` | LIVENESS | per-`find_capability`: TTL ≤ 48 ⟹ terminates. |
| `ext_cap_walk_ttl_bounded` | LIVENESS | per-`find_ext_capability`: TTL ≤ ((4096-0x100)/8) ⟹ terminates. |
| `ext_cap_walk_no_legacy` | INVARIANT | per-`find_ext_capability`: cfg_size ≤ 256 ⟹ returns 0. |
| `enable_cnt_refcounted` | INVARIANT | per-`enable_device` + `disable_device`: enable_cnt balanced; never negative. |
| `enable_device_idempotent` | INVARIANT | per-`enable_device`: enable_cnt>0 path skips hw writes. |
| `restore_state_clears_state_saved` | INVARIANT | per-`restore_state`: state_saved set false on completion. |
| `master_bit_only_set_when_enabled` | INVARIANT | per-`set_master`: enable_cnt > 0 precondition. |
| `acs_path_terminates_at_root_or_endpoint` | LIVENESS | per-`acs_path_enabled`: walk terminates at end or NULL parent. |
| `reset_function_save_restore_round_trip` | INVARIANT | per-`reset_function`: state_saved set → reset → state_saved cleared via `restore_state`. |
| `pme_list_no_double_insert` | INVARIANT | per-`pci_pme_active`: dev never on `pci_pme_list` twice. |
| `cap_save_buffer_no_double_alloc` | INVARIANT | per-`_pci_add_cap_save_buffer`: returns -EEXIST if cap already buffered. |
| `pci_dev_lock_no_deadlock` | INVARIANT | per-`pci_bus_lock`: in-order acquisition + per-bridge recursion → terminates. |

### Layer 2: TLA+

`drivers/pci/pci-core.tla`:
- Per-`pci_enable_device` + `pci_disable_device` ref-count concurrent.
- Per-`pci_save_state` + `pci_restore_state` interleaved with hot-reset.
- Per-`pci_set_power_state` D0 ↔ D3hot ↔ D3cold transitions.
- Per-`pci_reset_function` interleaved with `pci_enable_device` from concurrent driver-bind.
- Properties:
  - `safety_enable_cnt_nonneg` — `enable_cnt >= 0` always.
  - `safety_bus_master_implies_enabled` — `is_busmaster ⟹ enable_cnt > 0`.
  - `safety_state_saved_invariant` — `state_saved ⟺ saved_config_space valid`.
  - `safety_d3cold_implies_bridge_d3` — `current_state == PCI_D3cold ⟹ upstream_bridge.bridge_d3`.
  - `liveness_enable_eventually_completes` — `pci_enable_device` invocation terminates (modulo arch hook).
  - `liveness_reset_eventually_completes` — `pci_reset_function` terminates within `PCIE_RESET_READY_POLL_MS`.
  - `safety_acs_path_monotone` — adding an ACS-disabled upstream bridge can only decrement `acs_path_enabled` truth value.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `enable_device` pre: dev valid; post: enable_cnt incremented ∧ (enable_cnt was 0 ⟹ hw cmd-reg written) | `PciCore::enable_device` |
| `set_master` pre: enable_cnt > 0; post: `PCI_COMMAND.MASTER` set ∧ `dev.is_busmaster = true` | `PciCore::set_master` |
| `save_state` post: `state_saved = true` ∧ saved_config_space[0..16] = config[0..64] | `PciCore::save_state` |
| `restore_state` pre: `state_saved = true`; post: config[0..64] = saved_config_space[0..16] ∧ `state_saved = false` | `PciCore::restore_state` |
| `find_capability` post: returned offset ∈ [0x40, 0xFF] ∨ 0; no infinite loop | `PciCore::find_capability` |
| `find_ext_capability` post: returned offset ∈ [0x100, 0xFFC] ∨ 0; no infinite loop | `PciCore::find_ext_capability` |
| `acs_path_enabled` post: returned-true ⟺ every bridge on path has flags set | `PciCore::acs_path_enabled` |
| `pci_dev_lock` post: dev.dev.mutex held; `_unlock` post: released | `PciCore::dev_lock` |
| `enable_atomic_ops_to_root` post: returned-Ok ⟺ all upstream + dev support `cap_mask` ∧ `ATOMIC_EGRESS_BLOCK == 0` | `PciCore::enable_atomic_ops_to_root` |

### Layer 4: Verus/Creusot functional

`Per-driver bring-up: pci_enable_device → pci_set_master → pci_request_regions → pci_save_state → driver-DMA → pci_disable_device` semantic equivalence: PCIe Base Spec 5.0 §6.5 (Functional Reset), PCI 3.0 §6.2 (Configuration Space Header), PCIe Base Spec §7.5.3 (PCI Express Capability Structure), `Documentation/PCI/pci.rst`.

`Per-PM transition: D0 ↔ D3hot via pci_save_state + pci_set_power_state(D3hot) + pci_set_power_state(D0) + pci_restore_state` semantic equivalence: PCI PM Spec 1.2 §3.

`Per-Reset: pcie_flr round-trip` semantic equivalence: PCIe Base Spec §6.6 (Function-Level Reset).

`Per-ACS-path-traversal` semantic equivalence: PCIe Base Spec §6.12 (Access Control Services).

## Hardening

(Inherits row-1 features from `drivers/pci/00-overview.md` § Hardening.)

PCI core reinforcement:

- **Per-`enable_cnt` atomic ref-count** — defense against per-double-enable / per-double-disable that races MSI/MSI-X teardown with another driver-bind.
- **Per-cap-walk TTL bounded to 48 (std) / 480 (ext)** — defense against per-malformed-config-space infinite-loop (hostile guest emul + buggy hw).
- **Per-`pci_find_*_capability` config-read failure → 0** — defense against per-link-down config-read returning all-Fs (`PCIBIOS_DEVICE_NOT_FOUND`) crashing the walker.
- **Per-`pci_restore_config_space` cmd-register-last** — defense against per-stale-cmd-bits asserting MEM/IO/MASTER before BARs are restored, causing spurious DMA.
- **Per-`pci_dev_d3_sleep` honored on D-state transition** — defense against per-spec-required-D3-settling-time violation (1500 µs to D3hot, 10 ms from D3cold).
- **Per-`pci_bridge_d3_blacklist` DMI quirks** — defense against per-Apple/per-Dell firmware bug that hangs on bridge D3cold (real upstream bug from 2019–2024).
- **Per-`PCI_COMMAND.INTX_DISABLE` cleared only when pin != 0** — defense against per-spurious-INTx on MSI/MSI-X-only devices.
- **Per-`PCI_COMMAND.BUS_MASTER` cleared in `do_pci_disable_device`** — defense against per-stale-DMA after driver unload (e.g., kexec).
- **Per-`pci_init_reset_methods` priority order (FLR → AF-FLR → PM → bus → CXL)** — defense against per-untested-method-first that destabilizes downstream tree.
- **Per-`pci_dev_wait` 60 s ceiling on link-up after reset** — defense against per-non-responsive device causing reset path to hang kernel.
- **Per-`pci_acs_path_enabled` walks full path** — defense against per-IOMMU-group-isolation bypass (incomplete ACS check would let peer-DMA leak between groups).
- **Per-`pci_enable_atomic_ops_to_root` egress-block check** — defense against per-AtomicOp-translation hang.
- **Per-`pci_add_dma_alias` mask** — defense against per-DMA-source-id mismatch between bridge and behind-bridge device.
- **Per-`pci_setup` cmdline `nomsi` / `noaer` / `noats` / `noacs`** — defense against per-broken-hw-quirk via boot disable.
- **Per-`resource_alignment_param` parser** — defense against per-malformed-cmdline DoS.
- **Per-`pci_dev_lock` mandatory pre-reset** — defense against per-reset-races-driver-detach.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Bus enumeration / `pci_scan_root_bus` (covered in `probe.md` Tier-3)
- Driver matching / `pci_register_driver` (covered in `pci-driver.md` Tier-3 — sibling)
- Config-space transport / ECAM / CF8/CFC (covered in `access.md` Tier-3)
- MSI/MSI-X allocation (covered in MSI Tier-3 — `drivers/pci/msi/`)
- AER body / handler / EDR (covered in AER Tier-3 — `drivers/pci/pcie/aer.c`)
- DPC body (covered in DPC Tier-3 — `drivers/pci/pcie/dpc.c`)
- PME body / portdrv (covered in portdrv Tier-3)
- ASPM body (covered in ASPM Tier-3 — `drivers/pci/pcie/aspm.c`)
- SR-IOV body / VF lifecycle (covered in SRIOV Tier-3 — `drivers/pci/iov.c`)
- ATS / PRI / PASID body (covered in ATS Tier-3 — `drivers/pci/ats.c`)
- Quirks per-device-id workarounds (covered in `quirks.md` Tier-3 — `drivers/pci/quirks.c`)
- BAR sizing during enumeration (covered in `setup-res.md` + `setup-bus.md` Tier-3)
- Hotplug (covered in hotplug Tier-3 — `drivers/pci/hotplug/`)
- PCI sysfs surface body (covered in `pci-sysfs.md` Tier-3)
- VPD / DOE parsers (covered separately)
- p2pdma (covered separately — `drivers/pci/p2pdma.c`)
- Implementation code
