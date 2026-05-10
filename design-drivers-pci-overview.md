---
title: "Tier-2: drivers/pci — PCI/PCIe core (probe + capability scan + AER + SR-IOV + ATS/PASID + hotplug + ASPM + MSI/MSI-X)"
tags: ["tier-2", "drivers", "pci", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the PCI / PCIe / PCIe-CXL host stack. Sits as a `bus_type` consumer of `drivers/base/00-overview.md` (LDM); supplies devices to ~80% of in-tree drivers (NIC, NVMe, GPU, USB-host, AHCI, RAID HBA, sound, capture, accelerator, virtio-pci, vfio-pci, …). Owns:

- **Bus enumeration** (`probe.c` + `bus.c` + `setup-bus.c` + `setup-res.c` + `host-bridge.c` + `ecam.c`): root-bus discovery, recursive scan via type-1 bridges, BAR sizing + assignment, BIOS/firmware-resource takeover, secondary/subordinate bus number assignment, MMIO/IO/prefetch window assignment
- **Config-space access** (`access.c` + `vpd.c` + `doe.c`): PCI config-space read/write (legacy CF8/CFC port + ECAM mmio + ACPI MMCONFIG); VPD parsing; DOE (Data Object Exchange) for CMA/SPDM
- **Capability scan**: standard + extended capability iteration (PM, MSI, MSI-X, PCIe, AER, VC, SRIOV, ATS, PRI, PASID, ACS, LTR, PTM, DPC, ROM, DSN, MFVC, ARI, RCRB, etc.)
- **MSI / MSI-X** (`msi/`): interrupt allocation via msi_domain framework (cross-ref `kernel/irq/`)
- **Driver matching + binding** (`pci-driver.c`): `struct pci_driver` registration, `pci_device_id` table match, modalias `pci:v...d...sv...sd...bc...sc...i...`
- **AER** (`pcie/aer.c` + `pcie/aer_inject.c` + `pcie/aer_cxl_rch.c` + `pcie/edr.c`): Advanced Error Reporting — correctable + uncorrectable (non-fatal + fatal) handling, CPER/APEI integration, EDR (Error Disconnect Recovery)
- **DPC** (`pcie/dpc.c`): Downstream Port Containment — auto-disable downstream port on fatal error, with EDR-driven re-enumeration
- **PME** (`pcie/pme.c`): Power-Management Event handling
- **PTM** (`pcie/ptm.c`): Precision Time Measurement
- **bwctrl** (`pcie/bwctrl.c`): bandwidth/link-speed change notification
- **portdrv** (`pcie/portdrv.c`): the synthetic per-port-services driver that owns AER/PME/DPC/PTM/bwctrl as sub-services
- **rcec** (`pcie/rcec.c`): Root Complex Event Collector
- **tlp** (`pcie/tlp.c`): TLP log decoding for AER/DPC headers
- **ASPM** (`pcie/aspm.c`): Active State Power Management (L0s, L1, L1.1, L1.2 substates)
- **SR-IOV** (`iov.c`): Single-Root I/O Virtualization (PF + VF lifecycle, `sriov_numvfs` sysfs control)
- **ATS** (`ats.c`): Address Translation Services (ATS + PRI + PASID enable)
- **p2pdma** (`p2pdma.c`): peer-to-peer DMA between PCIe devices (NVMe-of, GPU-direct)
- **hotplug** (`hotplug/`): PCIe native hotplug (`pciehp`), ACPI hotplug (`acpiphp`), CPCI/PCI hotplug (`cpqphp`, `ibmphp`, `shpchp`, `cpcihp_*`)
- **controllers** (`controller/`): per-host-bridge driver glue (mostly ARM-SoC-specific; on x86_64 the ACPI/MMCONFIG code in `pci-acpi.c` is the host-bridge driver)
- **endpoint** (`endpoint/`): PCIe endpoint mode (when SoC acts as PCI device, not host) — out-of-scope for x86_64 v0
- **switch** (`switch/`): PCI switch enumeration support
- **PCI-ACPI** (`pci-acpi.c`): ACPI _PRT (PCI routing table) + _OSC (negotiation of OS-controlled features) + _DSM + ACPI-PCI hotplug glue
- **sysfs surface** (`pci-sysfs.c` + `pci-label.c` + `npem.c`): per-device sysfs files + LED-control + NPEM (Native PCIe Enclosure Management)
- **MMIO/MMAP** (`mmap.c` + `iomap.c`): user-mappable BARs (resource_files) + iomap helpers for drivers
- **Quirks** (`quirks.c`): per-device-id workarounds (broken-MSI, missing-acs, unaligned-MMIO, etc.) — large; keep maintained
- **bridge emulation** (`pci-bridge-emul.c`): software-emulated PCI-to-PCI bridge for ARM-SoC root-ports without real bridge HW
- **stub** (`pci-stub.c` + `pci-pf-stub.c` + `pci-mid.c` + `ide.c`): claim-and-do-nothing drivers for legacy IDE / SR-IOV PFs claimed by VFIO / Atom MID
- **devres** (`devres.c`): pcim_* managed-resource helpers (devm_* analog for PCI)
- **OF + of_property** (`of.c` + `of_property.c`): device-tree-described PCI buses (mostly ARM)
- **irq** (`irq.c`): legacy INTx routing aggregation

Heavily cross-referenced from every PCI-device driver Tier-2 (NVMe, AHCI, virtio-pci, mlx5, igb/ixgbe/i40e/ice, e1000e, r8169, drm/i915, drm/amdgpu, xhci-pci, snd-hda-intel, …), `kernel/irq/00-overview.md` (msi_domain consumer), `drivers/base/00-overview.md` (parent LDM), `drivers/iommu/00-overview.md` (IOMMU + ATS pairing), `drivers/vfio/00-overview.md` (vfio-pci is the PCI passthrough endpoint), `arch/x86/cpu-init.md` (MMCONFIG ECAM region BAR), `arch/x86/idt.md` (legacy INTx vectors).

### Out of Scope

- Per-Tier-3 (Phase D)
- ARM-SoC-specific PCI host-bridge controllers in `drivers/pci/controller/` beyond a generic-ECAM stub (out of v0 since x86_64-only)
- PCI endpoint-mode framework (`drivers/pci/endpoint/` — no x86_64 SoC ships as PCI-EP)
- CXL upper-layer (separate `drivers/cxl/` Tier-2 wrapper future)
- IOMMU drivers (separate `drivers/iommu/` Tier-2 wrapper future — `pci/ats.c` is the PCI-side glue only)
- VFIO drivers (separate `drivers/vfio/` Tier-2 wrapper future — pci-stub / pci-pf-stub is the PCI-side glue only)
- 32-bit-only paths
- Implementation code

### scope

This Tier-2 governs `/home/doll/linux-src/drivers/pci/` (~40 top-level files + `pcie/`, `msi/`, `hotplug/`, `controller/`, `endpoint/`, `switch/` subdirs) plus public headers `include/linux/pci.h`, `include/linux/pci-{acpi,doe,p2pdma,ats,ecam,epc,epf}.h`, UAPI `include/uapi/linux/pci.h` + `include/uapi/linux/pci_regs.h`.

### compatibility contract — outline

### Sysfs surface

Per-device sysfs at `/sys/bus/pci/devices/0000:BB:DD.F/`:
- Identity: `vendor`, `device`, `subsystem_vendor`, `subsystem_device`, `class`, `revision`, `irq`, `local_cpus`, `local_cpulist`, `numa_node`, `pools` (DMA), `aer_dev_correctable`, `aer_dev_fatal`, `aer_dev_nonfatal`
- Config space: `config` (raw 256B / 4KB binary file readable + writable with CAP_SYS_RAWIO)
- BAR resources: `resource` (text triplets), `resource0`/`resource0_resize`/`resource0_wc`/.../`resource5*` (each BAR mmap'able when prefetchable+write-combine ok), `rom`
- Power: `power/{control, autosuspend_delay_ms, runtime_status, ...}` (inherited from device-core), `d3cold_allowed`
- Hotplug: `remove`, `rescan`, `reset`, `reset_method`, `reset_subordinate`
- IOV: `sriov_totalvfs`, `sriov_numvfs` (write-rw), `sriov_offset`, `sriov_stride`, `sriov_vf_device`, `sriov_drivers_autoprobe`, `vendor`/`device` of VFs
- ATS / PRI / PASID: `ats`, `pri`, `pasid` (presence + enable state)
- AER: `aer_dev_correctable`, `aer_dev_fatal`, `aer_dev_nonfatal`, `aer_rootport_total_err_*`
- ASPM: `link/{l0s_aspm,l1_aspm,l1_1_aspm,l1_2_aspm,l1_1_pcipm,l1_2_pcipm,clkpm}` (write-rw)
- VPD: `vpd` (binary)
- Driver: `driver` symlink, `driver_override` (writable), `unbind`/`bind`/`uevent`/`modalias` (inherited from LDM)
- LED / enclosure: `npem`, `npem_locate`, `npem_fault`, `npem_rebuild`, `npem_pfa`, `npem_hotspare`, `npem_critical_array`, `npem_failed_array`, `npem_invalid_state`, `npem_disabled`
- Label / Index / Acpi-Index: `label`, `index`, `acpi_index`
- Misc: `current_link_speed`, `current_link_width`, `max_link_speed`, `max_link_width`, `secondary_bus_number`, `subordinate_bus_number`

Per-bus + per-class + per-host-bridge directories also identical.

Layout + content + permissions byte-identical so `lspci` / `setpci` / `lsblk` / `udevadm` / SRIOV-management tools work unchanged.

### `/proc/bus/pci/`

Legacy `/proc/bus/pci/devices` (line-per-device summary) + `/proc/bus/pci/<bus>/<dev.fn>` (raw config-space) byte-identical (still used by older `lspci -P0`).

### `lspci` modalias matching

`MODULE_DEVICE_TABLE(pci, ...)` produces `pci:v00008086d000010D3sv*sd*bc02sc00i00`-style alias. Format frozen — `depmod` + `modprobe` consume unchanged.

### `pci_dev->devfn` + bus-domain-segment numbering

Domain (PCI segment from ACPI MCFG) : Bus : Device.Function format identical (`0000:01:00.0`); `lspci -D` shows the same string set across all reachable buses.

### MSI / MSI-X allocation

`pci_alloc_irq_vectors()` API unchanged; per-vector `pci_irq_vector(pdev, n)` returns the same Linux IRQ numbers under the same allocation rules. msi_domain hierarchy identical (cross-ref `kernel/irq/`).

### AER reporting

`/sys/bus/pci/devices/.../aer_dev_*` counters byte-identical. Per-class AER report message format byte-identical (`pcieport 0000:00:1c.0: AER: ... Bad TLP, Bad DLLP, ...`); used by ras-utils + sosreport. CPER / EINJ injection paths via `apei` honored.

### SR-IOV control

`echo 8 > /sys/bus/pci/devices/0000:01:00.0/sriov_numvfs` enables 8 VFs. Per-VF appears as `0000:01:00.1` ... `0000:01:01.0` etc. `sriov_drivers_autoprobe` controls whether VFs auto-bind their driver. Wire format identical so `mlxconfig` / Intel-iavf-driver-binder / etc. work unchanged.

### ATS / PASID enable + binding

`pci_enable_ats()` / `pci_enable_pasid()` / `pci_enable_pri()` API identical. IOMMU integration via `arm_smmu_v3` / `intel_iommu` / `amd_iommu`; consumer side behaves the same so passthrough + DMA-translation works unchanged.

### Hotplug

`pciehp` driver auto-loads for native PCIe hotplug-capable downstream ports. Slot files at `/sys/bus/pci/slots/<slot>/{address,attention,latch,power,adapter,test,max_bus_speed,cur_bus_speed}`. Wire format byte-identical so hotplug tooling (`pciehp_force=1` cmdline + manual `echo 1 > power` / `echo 0 > power`) works unchanged.

### ASPM control

`/sys/module/pcie_aspm/parameters/policy` (default / performance / powersave / powersupersave) + per-link `/sys/bus/pci/devices/.../link/*_aspm` writable controls. Defaults inherited from BIOS + ACPI _OSC negotiation. Wire format identical.

### `/sys/bus/pci/rescan` + `/sys/bus/pci/devices/.../rescan`

Triggers a partial or full PCI re-scan. Writable by root. Behavior identical.

### `pci=` command-line parameters

A long list of `pci=...` cmdline options must parse + behave identically: `nomsi`, `noacpi`, `nocrs`, `noaer`, `realloc`, `routeirq`, `assign-busses`, `bigroot_window=`, `cbiosize=`, `cbmemsize=`, `compaq_nofb`, `firmware`, `pcie_bus_safe`/`tune_off`/`peer2peer`/`perf`, `nobios`, `noearly`, `disable_acs_redir=`, etc. Documented set MUST be parsed identically.

### Quirks

`drivers/pci/quirks.c` is ~9000 lines of per-device workarounds. The complete table MUST be carried over (modulo workarounds for HW that genuinely never exists outside ARM SoCs that we don't support). Quirk-application order + at-which-stage (early / header / final / resume) is part of the contract.

### Endpoint mode

Out of v0 scope (no x86_64 SoC needs PCI-EP mode in tree).

### `pci_save_state()` / `pci_restore_state()` semantics

For runtime + system PM. State-save list (config-space + MSI/MSI-X + ASPM + AER + LTR + ACS + each capability) MUST match upstream. Resume order via dpm_list (cross-ref `drivers/base/pm-system.md`).

### CXL coexistence

CXL devices (Type 1/2/3) appear as PCIe devices first. PCIe core hands off to `drivers/cxl/` (separate Tier-2 future) once CXL DVSEC is detected. Coexistence handoff API (`pci_find_dvsec_capability(PCI_VENDOR_ID_CXL, ...)`) MUST work unchanged.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/pci/probe.md` | `probe.c`: bus enumeration + recursive scan + device creation |
| `drivers/pci/setup.md` | `setup-bus.c` + `setup-res.c` + `host-bridge.c`: BAR sizing + assignment + bus-window assignment |
| `drivers/pci/access.md` | `access.c` + `ecam.c`: config-space accessors (CF8/CFC + MMCONFIG ECAM) |
| `drivers/pci/pci-driver.md` | `pci-driver.c`: pci_driver registration + match-table + bind/unbind glue |
| `drivers/pci/sysfs.md` | `pci-sysfs.c` + `pci-label.c` + `npem.c`: per-device sysfs surface |
| `drivers/pci/quirks.md` | `quirks.c`: per-device workaround table (carry verbatim) |
| `drivers/pci/devres.md` | `devres.c`: pcim_* managed resources |
| `drivers/pci/mmap.md` | `mmap.c` + `iomap.c`: user-mappable BARs + iomap helpers |
| `drivers/pci/iov.md` | `iov.c`: SR-IOV PF + VF lifecycle |
| `drivers/pci/ats.md` | `ats.c`: ATS + PRI + PASID enable |
| `drivers/pci/p2pdma.md` | `p2pdma.c`: peer-to-peer DMA |
| `drivers/pci/doe.md` | `doe.c`: Data Object Exchange (CMA / SPDM) |
| `drivers/pci/vpd.md` | `vpd.c`: Vital Product Data parsing |
| `drivers/pci/pci-acpi.md` | `pci-acpi.c`: ACPI _PRT + _OSC negotiation + ACPI hotplug glue |
| `drivers/pci/msi.md` | `msi/{api,irqdomain,legacy,msi,pcidev_msi}.c`: MSI / MSI-X allocation, msi_domain hierarchy |
| `drivers/pci/pcie-portdrv.md` | `pcie/portdrv.c`: per-port services driver |
| `drivers/pci/pcie-aer.md` | `pcie/aer.c` + `aer_inject.c` + `aer_cxl_rch.c` + `tlp.c` + `edr.c`: AER + EDR + TLP-log + injection |
| `drivers/pci/pcie-dpc.md` | `pcie/dpc.c`: Downstream Port Containment |
| `drivers/pci/pcie-pme.md` | `pcie/pme.c`: PME handling |
| `drivers/pci/pcie-ptm.md` | `pcie/ptm.c`: Precision Time Measurement |
| `drivers/pci/pcie-aspm.md` | `pcie/aspm.c`: Active State Power Management |
| `drivers/pci/pcie-bwctrl.md` | `pcie/bwctrl.c`: bandwidth-change notifier |
| `drivers/pci/pcie-rcec.md` | `pcie/rcec.c`: Root Complex Event Collector |
| `drivers/pci/hotplug-pciehp.md` | `hotplug/pciehp_*.c`: native PCIe hotplug |
| `drivers/pci/hotplug-acpiphp.md` | `hotplug/acpiphp_*.c`: ACPI hotplug |
| `drivers/pci/hotplug-shpc.md` | `hotplug/shpchp_*.c`: SHPC (PCI-X hotplug) |
| `drivers/pci/bridge-emul.md` | `pci-bridge-emul.c`: software bridge emulation |
| `drivers/pci/stub.md` | `pci-stub.c` + `pci-pf-stub.c`: claim-and-do-nothing drivers |
| `drivers/pci/of.md` | `of.c` + `of_property.c`: device-tree PCI nodes (ARM-mostly; x86 stubs OK) |
| `drivers/pci/irq.md` | `irq.c`: legacy INTx aggregation |

### compatibility outline (top-level)

- REQ-O1: All `/sys/bus/pci/...` sysfs files + `/proc/bus/pci/...` legacy files byte-identical (lspci / setpci / udevadm / sosreport consume unchanged).
- REQ-O2: Config-space access via CF8/CFC + ACPI MMCONFIG ECAM byte-identical (raw `config` file readable yields identical bytes).
- REQ-O3: `MODULE_DEVICE_TABLE(pci, ...)` modalias format byte-identical (`pci:v...d...sv...sd...bc...sc...i...`).
- REQ-O4: SR-IOV control (`sriov_numvfs`, `sriov_drivers_autoprobe`, …) byte-identical.
- REQ-O5: AER report message format + per-device counters byte-identical.
- REQ-O6: PCIe native hotplug (pciehp) + ACPI hotplug (acpiphp) sysfs slot surface byte-identical.
- REQ-O7: ASPM policy module-param + per-link sysfs control byte-identical.
- REQ-O8: `pci=` cmdline option set parsed identically.
- REQ-O9: `pci_dev` API for in-tree drivers source-compatible: `pci_enable_device`, `pci_request_regions`, `pci_set_master`, `pci_alloc_irq_vectors`, `pci_irq_vector`, `pci_save_state`, `pci_restore_state`, `pci_enable_msi`, `pci_disable_msi`, `pci_enable_msix_range`, `pci_alloc_consistent`, `pci_map_single`, `pcim_*`, `pci_register_driver`, `pci_iomap`, `pci_resource_*`, `pci_find_capability`, `pci_find_ext_capability`, `pci_dev_present`, `pci_for_each_dma_alias`.
- REQ-O10: Quirks table carried over verbatim with same application stage (PCI_FIXUP_EARLY / _HEADER / _FINAL / _ENABLE / _RESUME / _SUSPEND).
- REQ-O11: ATS / PASID / PRI enable APIs identical; IOMMU integration unchanged.
- REQ-O12: TLA+ models declared at this Tier-2 (BAR-window assignment correctness, AER recovery state machine, hotplug slot state machine, SRIOV VF enable/disable race-freedom).
- REQ-O13: Verus/Creusot Layer-4 functional contracts on config-space access (no read past 4 KB extended config; no write to RO bits violating spec) + on capability-iteration termination (no infinite-loop on malformed cap-list).
- REQ-O14: Hardening: row-1 features applied per `00-security-principles.md`, with PCI-specific reinforcement (config-space write CAP_SYS_RAWIO + LSM hooks, BAR mmap LSM mediation, driver_override LSM mediation).

### acceptance criteria (top-level)

- [ ] AC-O1: `lspci -vvv` against a reference machine produces output byte-identical to upstream baseline. (covers REQ-O1, REQ-O2)
- [ ] AC-O2: `udevadm info --query=property /sys/bus/pci/devices/0000:00:1f.3` returns identical property set. (covers REQ-O3)
- [ ] AC-O3: NVMe drive enumerates + binds to `nvme` driver; `nvme list` shows the namespace. (covers REQ-O9)
- [ ] AC-O4: `echo 4 > /sys/bus/pci/devices/.../sriov_numvfs` on an SRIOV-capable NIC creates 4 VFs that enumerate. (covers REQ-O4)
- [ ] AC-O5: AER injection via `setpci` + EINJ produces correct AER report message + counter increment. (covers REQ-O5)
- [ ] AC-O6: PCIe-hotplug add/remove via `pciehp` (e.g., physical hot-add) updates `/sys/bus/pci/slots/.../adapter` + triggers `add@`/`remove@` uevents. (covers REQ-O6)
- [ ] AC-O7: `cat /sys/module/pcie_aspm/parameters/policy` returns same default as upstream; per-link write toggles ASPM state. (covers REQ-O7)
- [ ] AC-O8: `pci=nomsi` cmdline disables MSI globally; `pci_alloc_irq_vectors` falls back to legacy INTx. (covers REQ-O8)
- [ ] AC-O9: vfio-pci passthrough via `driver_override = vfio-pci` + qemu `-device vfio-pci,host=...` works. (covers REQ-O9, REQ-O11)
- [ ] AC-O10: Suspend-to-RAM + resume on a reference laptop preserves PCI config-space state for every device. (covers REQ-O9 — pci_save_state / restore_state)
- [ ] AC-O11: kselftest `tools/testing/selftests/pci_endpoint/` skips on x86_64 (out of v0 scope), other PCI selftests pass. (covers REQ-O12)
- [ ] AC-O12: SPDK / DPDK userspace driver running on `vfio-pci` works unchanged (the entire BAR-mmap + interrupt-eventfd path). (covers REQ-O1, REQ-O9, REQ-O11)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/pci/bar_assignment.tla` | `drivers/pci/setup.md` (proves: recursive bus-window assignment terminates and produces non-overlapping BAR placements satisfying alignment + prefetch-window constraints; `pci=realloc` retry path converges) |
| `models/pci/aer_recovery.tla` | `drivers/pci/pcie-aer.md` (proves: AER state machine — DETECT → DISABLE → SLOT_RESET → RESUME — under fatal error; concurrent DPC trigger + AER report coordinated correctly; per-driver `error_handlers` callbacks invoked exactly once per phase) |
| `models/pci/hotplug_slot.tla` | `drivers/pci/hotplug-pciehp.md` (proves: pciehp slot state machine — EMPTY → CARD_PRESENT → POWERED → ENABLED → ... — under concurrent presence-change + power-button-press + sysfs `power` write; no missed event causes desync) |
| `models/pci/sriov_lifecycle.tla` | `drivers/pci/iov.md` (proves: VF enable/disable + driver-autoprobe interleaving — concurrent `sriov_numvfs` write while VFs being probed never produces partial state) |
| `models/pci/cap_iteration.tla` | `drivers/pci/access.md` (proves: standard + extended capability list iteration always terminates given any 4 KB config-space content — defense against malformed/malicious caps causing infinite loop) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/pci/access.md` | `pci_read_config_*` / `pci_write_config_*` pre/post-condition: offset bounds-checked (< 256 for legacy, < 4096 for extended); ECAM mmio path validates segment+bus+devfn before dereference |
| `drivers/pci/pci-driver.md` | `pci_register_driver` invariant: per-driver pci_device_id table sentinel-terminated; match returns at-most-one entry |
| `drivers/pci/iov.md` | `pci_enable_sriov(pdev, nr)` post-condition: nr VFs created with consecutive devfn values + correct VF-only fields; partial failure rolls back |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-pci_dev refcount (via embedded `struct device`) + per-host-bridge refcount + per-slot refcount use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | `struct pci_driver`, `pci_device_id` tables, per-controller `pci_ops` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | BAR-resource arithmetic (start + size + alignment) + bus-window assignment uses checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-controller pci_ops + per-port portdrv_ops vtables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed pci_dev + per-slot state cleared (carries vendor / device id used by some downstream consumers via stale pointer) | § Default-on configurable off |
| **BARRIER** | config-space writers fence MMIO write-completion before subsequent reads (already true in upstream via `pci_*` accessors but Rookery makes it explicit) | § Mandatory |

### Row-2 (LSM-stackable) features for PCI

LSM hooks called: file-LSM hooks on every sysfs write (`config`, `driver_override`, `bind`, `unbind`, `sriov_numvfs`, `remove`, `rescan`, `reset`, `link/*_aspm`). SELinux + AppArmor + future GR-RBAC stackable LSM each mediate.

GR-RBAC adds:
- Per-role disallow `config` raw write (defends against driver attacks via setpci).
- Per-role disallow `driver_override` write (defends against VFIO-rebind container escapes).
- Per-role disallow `sriov_numvfs` write (only network admin can enable VFs).
- Per-role disallow `remove`/`rescan`/`reset` (only kernel admin).
- Per-role audit of all BAR mmap calls.

### PCI-specific reinforcement

- **Capability list iteration bounded**: every `pci_find_capability` / `pci_find_ext_capability` walks at most N steps (N = 48 for std caps, N = 1024 for ext caps); malformed device cannot DoS by producing a cap-list cycle (Layer-2 `cap_iteration.tla` is the proof).
- **Config-space write CAP_SYS_RAWIO**: writes to `/sys/.../config` require CAP_SYS_RAWIO in init userns (not user-namespace forgeable).
- **BAR mmap requires CAP_SYS_RAWIO** plus LSM mediation (existing upstream behavior; Rookery adds LSM hook explicit).
- **AER injection facility (`aer_inject.c`)** behind CONFIG_PCIEAER_INJECT default-N + CAP_SYS_ADMIN gate.
- **VPD writes** require CAP_SYS_RAWIO + LSM mediation; some VPD regions are write-once + can brick devices.
- **Quirks application**: every quirk that writes config-space documents the writeback's affected bits + rationale comment; Rookery preserves these (no hidden quirks).
- **Hotplug power-on cap**: per-port pciehp `power` write (`echo 1 > .../power`) honored only when `slot_capable + presence_detected`.
- **DPC + EDR trigger**: on uncorrectable fatal error, link is contained per spec (don't silently continue).
- **ASPM disable-on-AER-shower**: if AER correctable-error rate exceeds threshold, ASPM auto-disabled (defense against degraded-link error storms).

