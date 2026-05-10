# Tier-2: drivers/vfio — VFIO (userspace device passthrough: vfio-pci + iommufd + group / container / mdev)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/vfio/
  - include/linux/vfio.h
  - include/uapi/linux/vfio.h
-->

## Summary

Tier-2 overview for VFIO — the kernel framework that lets userspace drivers (qemu-system-x86_64 device passthrough, DPDK, SPDK, custom accelerator userspace stacks) drive a hardware device directly while DMA is still safely contained by an IOMMU. VFIO is the consumer side of `drivers/iommu/00-overview.md` and the producer side of the userspace UAPI — it's the third vertex of the KVM ↔ IOMMU ↔ VFIO passthrough triangle, the substrate qemu uses for `vfio-pci`, the substrate DPDK uses for kernel-bypass userspace networking, and the substrate every modern accelerator userspace driver builds on (Intel-IDXD-cdev, NVIDIA-MIG via vfio-mdev, vfio-fsl-mc, vfio-platform).

Two pluggable layers:

- **Bus drivers** (one per bus type): `vfio-pci-core` (the bulk; `vfio_pci.c` + `vfio_pci_core.c` + bus-specific config-space + IGD + IRQ + R/W + DMA-buf), `vfio-platform` (DT-described platform devices), `vfio-fsl-mc` (Freescale MC bus), `vfio-cdx` (AMD CDX), `vfio-mdev` (mediated devices: a synthetic VF-like device implemented by an in-kernel driver — used by NVIDIA vGPU, Intel-GVT-g, S390 ccw/AP), and per-vendor variant drivers for live-migration support (`mlx5`, `pds`, `qat`, `hisilicon`, `nvgrace-gpu`, `xe`, `ism`, `virtio`).
- **IOMMU backends**: legacy "container" interface (`vfio_iommu_type1.c` + `container.c`) where the user opens `/dev/vfio/<group>` then attaches it to a `/dev/vfio/vfio` container, and the modern iommufd path (`iommufd.c`) where the user opens `/dev/iommu` directly + binds devices to it; both supported simultaneously for transition window.

Heavily cross-referenced from `virt/kvm/00-overview.md` (KVM↔VFIO IRQ bypass via `kvm_vfio.c` + posted-interrupts), `drivers/iommu/00-overview.md` (DMA isolation backbone), `drivers/pci/00-overview.md` (vfio-pci bus binding via `driver_override = vfio-pci`), `arch/x86/idt.md` (eventfd-backed user-mode interrupt delivery), `kernel/cgroup/00-overview.md` (per-VM resource accounting).

## Components

- **vfio core** (`vfio_main.c` + `vfio.h`): top-level subsystem init + `/dev/vfio/vfio` chardev (legacy container) + group registration + bus-driver registration
- **group** (`group.c`): per-iommu-group `/dev/vfio/<N>` chardev — open the group fd, get device fds via `VFIO_GROUP_GET_DEVICE_FD`
- **container** (`container.c`): legacy `vfio_container` object + IOMMU-driver registration (vfio-iommu-type1 or vfio-iommu-spapr-tce on PPC)
- **device cdev** (`device_cdev.c`): the modern per-device chardev `/dev/vfio/devices/vfio<N>` — bypass groups; bind directly to iommufd
- **iommufd backend** (`iommufd.c`): vfio-side glue connecting a vfio device to an `/dev/iommu` IOAS instead of legacy container
- **vfio_iommu_type1** (`vfio_iommu_type1.c`): the legacy IOMMU backend (uses `iommu_map` directly through the generic IOMMU API; consumes IOVA allocator)
- **vfio_iommu_spapr_tce** (`vfio_iommu_spapr_tce.c`): PowerPC sPAPR TCE backend (out of v0)
- **virqfd** (`virqfd.c`): per-IRQ eventfd binding — when a hardware IRQ fires, signal an eventfd userspace is listening on; replaces the kernel's normal IRQ-handler dispatch
- **debugfs** (`debugfs.c`): `/sys/kernel/debug/vfio/`
- **vfio-pci** (`pci/vfio_pci.c` + `pci/vfio_pci_core.c` + `pci/vfio_pci_config.c` + `pci/vfio_pci_intrs.c` + `pci/vfio_pci_rdwr.c` + `pci/vfio_pci_igd.c` + `pci/vfio_pci_zdev.c` + `pci/vfio_pci_dmabuf.c`): the PCI bus driver — covers config-space virtualization (some bits readable, some emulated, some hidden), BAR mmap pass-through, MSI/MSI-X virtualization with eventfd backing, INTx INTACK virtualization, IGD opregion (Intel iGPU passthrough), s390 zPCI features, dmabuf export of BAR memory
- **vfio-mdev** (`mdev/`): mediated-device framework — a parent driver claims a real device + spawns N "mediated" child devices each with its own UUID; each mediated child appears as a vfio device to userspace. Used by NVIDIA-vGPU, Intel-GVT-g, S390-ccw/AP, KVM-PCI-VGA-card-on-virtual-machine.
- **vfio-platform** (`platform/`): out of v0 (ARM-SoC platform-bus passthrough; no x86_64 use case beyond mdev)
- **vfio-fsl-mc** (`fsl-mc/`): out of v0 (Freescale MC bus, ARM-only)
- **vfio-cdx** (`cdx/`): out of v0 (AMD CDX bus, ARM-only)
- **vendor variant drivers** (`pci/{mlx5,pds,qat,hisilicon,nvgrace-gpu,xe,ism,virtio}`): per-vendor extensions of `vfio-pci-core` providing live-migration state save/restore (the basic vfio-pci doesn't support migration; vendor variant drivers add the device-specific snapshot/restore + dirty-page tracking).

## Scope

This Tier-2 governs `/home/doll/linux-src/drivers/vfio/` (~10 top-level files + `pci/`, `mdev/`, `platform/`, `fsl-mc/`, `cdx/` subdirs), public headers `include/linux/vfio.h` + `include/linux/vfio_pci_core.h`, UAPI `include/uapi/linux/vfio.h`. Platform / FSL-MC / CDX subdirs compile-gated out for v0; PCI + mdev + iommufd backend in active maintenance.

## Compatibility contract — outline

### `/dev/vfio/vfio` (legacy container) ioctl wire format

ioctl numbers + struct layouts byte-identical:
- `VFIO_GET_API_VERSION`, `VFIO_CHECK_EXTENSION`, `VFIO_SET_IOMMU` (set IOMMU type: VFIO_TYPE1_IOMMU / VFIO_TYPE1v2_IOMMU / VFIO_NOIOMMU_IOMMU / VFIO_SPAPR_TCE_IOMMU / VFIO_TYPE1_NESTING_IOMMU)
- `VFIO_IOMMU_GET_INFO`, `VFIO_IOMMU_MAP_DMA`, `VFIO_IOMMU_UNMAP_DMA`, `VFIO_IOMMU_DIRTY_PAGES`

### `/dev/vfio/<N>` (group) ioctl wire format

- `VFIO_GROUP_GET_STATUS`, `VFIO_GROUP_SET_CONTAINER`, `VFIO_GROUP_UNSET_CONTAINER`, `VFIO_GROUP_GET_DEVICE_FD`

### `/dev/vfio/devices/vfio<N>` (modern cdev) ioctl wire format

- `VFIO_DEVICE_BIND_IOMMUFD` (bind to `/dev/iommu`), `VFIO_DEVICE_ATTACH_IOMMUFD_PT` / `VFIO_DEVICE_DETACH_IOMMUFD_PT` (attach/detach to a specific HWPT)

### Per-device-fd ioctls (work on group-derived fd OR cdev-derived fd)

- `VFIO_DEVICE_GET_INFO` (region count, IRQ count, flags)
- `VFIO_DEVICE_GET_REGION_INFO` (per-region: index, offset, size, flags, cap chain — VFIO_REGION_INFO_CAP_SPARSE_MMAP, VFIO_REGION_INFO_CAP_TYPE, VFIO_REGION_INFO_CAP_MSIX_MAPPABLE, …)
- `VFIO_DEVICE_GET_IRQ_INFO`, `VFIO_DEVICE_SET_IRQS` (eventfd binding for INTx / MSI / MSI-X / REQ / ERR)
- `VFIO_DEVICE_RESET`
- `VFIO_DEVICE_GET_PCI_HOT_RESET_INFO`, `VFIO_DEVICE_PCI_HOT_RESET`
- `VFIO_DEVICE_QUERY_GFX_PLANE`, `VFIO_DEVICE_GET_GFX_DMABUF` (mdev-vGPU display)
- `VFIO_DEVICE_IOEVENTFD`
- `VFIO_DEVICE_FEATURE` (multiplexer for migration / DMA-logging / per-device features: VFIO_DEVICE_FEATURE_MIGRATION / _DMA_LOGGING_START / _STOP / _REPORT / _MIG_DEVICE_STATE / _LOW_POWER_ENTRY / _LOW_POWER_EXIT / _PCI_VF_TOKEN / _BUS_MASTER / _PCI_PM)

Wire format byte-identical including reserved bytes.

### Region access (read / write / mmap)

Each device exposes N regions (PCI: BARs 0-5 + ROM + config-space + VGA), each at a per-region file-offset within the device fd:
- `read(devfd, buf, sz, offset = region_offset + reg_addr)` / `write(devfd, ...)`: emulated config-space access (with vfio masking); MMIO/IO BAR access
- `mmap(devfd, ..., offset = region_offset)`: direct mapping of BAR (when SPARSE_MMAP cap allows); user mmap'd address sees real BAR MMIO

Behavior byte-identical so qemu's vfio-pci device emulation reads/writes the same bytes.

### IRQ delivery via eventfd

`VFIO_DEVICE_SET_IRQS` with `VFIO_IRQ_SET_DATA_EVENTFD` registers an eventfd per-vector. When hardware IRQ fires, virqfd writes 1 to the eventfd. qemu typically pairs each eventfd with a KVM_IRQFD so the guest IRQ is delivered without userspace round-trip. Wire identical.

### `/sys/bus/pci/devices/.../vfio-dev/`

vfio-pci-bound devices expose `/sys/bus/pci/devices/<dev>/vfio-dev/vfio<N>/` with attributes `name`, `dev`, `migration/`, etc. Layout byte-identical.

### `/dev/vfio/devices/`

Per-device cdev `/dev/vfio/devices/vfio<N>` byte-identical naming + numbering. Major number 234 (dynamic but stable in upstream); minor number == `dev_set->dev_id`.

### `/sys/class/vfio_dev/`

Per-vfio-device class entries identical.

### Per-mdev sysfs

Mediated-device framework exposes per-parent `/sys/class/mdev_bus/.../mdev_supported_types/<type>/{available_instances, create, name, description, device_api}`. Wire-format byte-identical so `mdevctl` works unchanged.

### NOIOMMU mode

`VFIO_NOIOMMU_IOMMU` for trusted single-tenant scenarios where DMA isolation isn't required (DPDK on bare-metal). Requires CONFIG_VFIO_NOIOMMU=y + CAP_SYS_RAWIO + writes "1" to `/sys/module/vfio/parameters/enable_unsafe_noiommu_mode`. Behavior identical (not enabled by default; explicit opt-in).

### Live-migration UAPI

`VFIO_DEVICE_FEATURE_MIGRATION` reports per-device migration support. State machine: STOP → STOP_COPY → RESUMING → RUNNING (+ PRE_COPY / PRE_COPY_P2P optional). Per-vendor variant drivers implement state-save/restore + DMA dirty-page tracking. UAPI byte-identical so qemu live-migration with vfio-pci passthrough works unchanged across vendors that have variant driver support.

### vfio-pci config-space virtualization

Some config-space bits are passthrough (vendor/device id, BARs); some are emulated (PCIe link control, command register bits, ASPM, hot-reset state); some are hidden (PASID enable, SR-IOV control on the PF the user has). The complete bit-by-bit table in `vfio_pci_config.c` MUST match upstream so guest sees the right register values.

### IGD opregion + LPC bridge

`vfio_pci_igd.c` exposes Intel iGPU's ACPI OpRegion + host-bridge config-space copy as additional regions on the device fd. Required for Intel-GVT-d (full-PCI passthrough of integrated GPU). Wire format identical.

### dmabuf export (`vfio_pci_dmabuf.c`)

VFIO-PCI BAR can be exported as a dmabuf for cross-driver buffer sharing (e.g., GPU consuming SmartNIC-DMAR'd buffer). UAPI identical.

### s390 zPCI features (`vfio_pci_zdev.c`)

s390-specific (out of v0 — keep code present but compile-gated off).

## Tier-3 docs governed by this Tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/vfio/vfio-core.md` | `vfio_main.c` + `vfio.h`: subsystem init + bus-driver registration + group registration |
| `drivers/vfio/group.md` | `group.c`: legacy `/dev/vfio/<N>` group fd + `VFIO_GROUP_GET_DEVICE_FD` |
| `drivers/vfio/container.md` | `container.c`: legacy `/dev/vfio/vfio` container + IOMMU-driver registration |
| `drivers/vfio/device-cdev.md` | `device_cdev.c`: modern per-device `/dev/vfio/devices/vfio<N>` chardev |
| `drivers/vfio/iommufd-backend.md` | `iommufd.c`: vfio-side glue connecting a vfio device to an iommufd IOAS |
| `drivers/vfio/iommu-type1.md` | `vfio_iommu_type1.c`: legacy IOMMU backend (iommu_map via generic IOMMU API) |
| `drivers/vfio/virqfd.md` | `virqfd.c`: per-IRQ eventfd-backed signal |
| `drivers/vfio/debugfs.md` | `debugfs.c`: `/sys/kernel/debug/vfio/` |
| `drivers/vfio/pci-core.md` | `pci/vfio_pci.c` + `pci/vfio_pci_core.c` + `pci/vfio_pci_priv.h`: vfio-pci bus driver |
| `drivers/vfio/pci-config.md` | `pci/vfio_pci_config.c`: config-space virtualization (passthrough / emulated / hidden) |
| `drivers/vfio/pci-intrs.md` | `pci/vfio_pci_intrs.c`: INTx / MSI / MSI-X virtualization |
| `drivers/vfio/pci-rdwr.md` | `pci/vfio_pci_rdwr.c`: BAR + region read/write paths |
| `drivers/vfio/pci-igd.md` | `pci/vfio_pci_igd.c`: Intel iGPU OpRegion + LPC bridge regions |
| `drivers/vfio/pci-dmabuf.md` | `pci/vfio_pci_dmabuf.c`: dmabuf export of BAR memory |
| `drivers/vfio/mdev.md` | `mdev/`: mediated-device framework |
| `drivers/vfio/pci-mlx5-vmig.md` | `pci/mlx5/`: Mellanox ConnectX live-migration variant driver |
| `drivers/vfio/pci-pds-vmig.md` | `pci/pds/`: AMD Pensando live-migration variant |
| `drivers/vfio/pci-qat-vmig.md` | `pci/qat/`: Intel QAT live-migration variant |
| `drivers/vfio/pci-hisilicon-vmig.md` | `pci/hisilicon/`: HiSilicon ACC live-migration variant |
| `drivers/vfio/pci-nvgrace-vmig.md` | `pci/nvgrace-gpu/`: NVIDIA Grace+Hopper coherent-GPU passthrough |
| `drivers/vfio/pci-xe-vmig.md` | `pci/xe/`: Intel Xe SR-IOV VF migration |
| `drivers/vfio/pci-virtio-vmig.md` | `pci/virtio/`: virtio-PCI live-migration variant |

## Compatibility outline (top-level)

- REQ-O1: All `VFIO_*` ioctl numbers + struct layouts byte-identical to upstream UAPI (qemu / cloud-hypervisor / firecracker / DPDK / SPDK run unmodified).
- REQ-O2: Both legacy group-container path AND modern iommufd-cdev path supported simultaneously; existing qemu-using-group code paths AND new qemu-using-iommufd code paths both work.
- REQ-O3: vfio-pci config-space virtualization bit-table byte-identical to upstream (guest sees same register values).
- REQ-O4: BAR mmap + region read/write + IRQ-eventfd delivery semantics byte-identical.
- REQ-O5: Live-migration UAPI byte-identical for vendor variant drivers in tree (mlx5 / pds / qat / hisilicon / nvgrace / xe / virtio / ism).
- REQ-O6: vfio-mdev framework UAPI + sysfs surface byte-identical (mdevctl / nvidia-vgpu / intel-gvt-g run unmodified).
- REQ-O7: NOIOMMU mode opt-in identical (CONFIG_VFIO_NOIOMMU=y + module param + CAP_SYS_RAWIO).
- REQ-O8: Per-device sysfs `/sys/bus/pci/devices/.../vfio-dev/` + `/sys/class/vfio_dev/` byte-identical.
- REQ-O9: dmabuf export of BAR identical for cross-driver buffer-sharing consumers.
- REQ-O10: KVM↔VFIO IRQ-bypass posted-interrupt path identical (qemu pairs eventfd with KVM_IRQFD; bypass kicks in when supported).
- REQ-O11: TLA+ models declared at this Tier-2 (eventfd-backed IRQ delivery, group-container attach race-freedom, device-fd close while in-flight DMA, migration state machine).
- REQ-O12: Verus/Creusot Layer-4 functional contracts on vfio device-fd refcount + region-read/write bounds + IRQ-eventfd lifetime.
- REQ-O13: Hardening: row-1 features applied per `00-security-principles.md`, with VFIO-specific reinforcement (NOIOMMU mode default-off + audit-logged, vfio-pci config-space sanitization on every read/write, MSI-X table-mmap denied unless VFIO_REGION_INFO_CAP_MSIX_MAPPABLE set + IOMMU-validates).

## Acceptance Criteria (top-level)

- [ ] AC-O1: qemu `-device vfio-pci,host=0000:01:00.0` (legacy group mode) works on a passthrough NIC; guest sends/receives traffic. (covers REQ-O1, REQ-O2, REQ-O3, REQ-O4)
- [ ] AC-O2: qemu `-object iommufd,id=iommufd0 -device vfio-pci,host=...,iommufd=iommufd0` (modern iommufd-cdev mode) works equivalently. (covers REQ-O1, REQ-O2)
- [ ] AC-O3: DPDK testpmd against vfio-pci-bound NIC achieves line-rate on 25/100GbE reference HW. (covers REQ-O1, REQ-O4)
- [ ] AC-O4: SPDK NVMe userspace driver against vfio-pci-bound NVMe achieves IOPS within 2% of upstream baseline. (covers REQ-O1, REQ-O4)
- [ ] AC-O5: NVIDIA-vGPU mdev creation: `echo $UUID > /sys/class/mdev_bus/.../create` succeeds + qemu binds the mdev. (covers REQ-O6)
- [ ] AC-O6: mlx5-VF live-migration: qemu migrate of VM with passed-through ConnectX VF completes; guest network connectivity preserved. (covers REQ-O5)
- [ ] AC-O7: KVM-VFIO IRQ-bypass: posted-interrupt delivery latency to guest under load matches upstream baseline (per `perf kvm stat`). (covers REQ-O10)
- [ ] AC-O8: NOIOMMU mode: with `enable_unsafe_noiommu_mode=1` + DPDK testpmd works on bare-metal. (covers REQ-O7)
- [ ] AC-O9: GPU↔SmartNIC dmabuf sharing test: vfio-pci BAR exported as dmabuf, imported by drm consumer. (covers REQ-O9)
- [ ] AC-O10: kselftest `tools/testing/selftests/vfio/` passes. (covers REQ-O11, REQ-O12)
- [ ] AC-O11: ACS-isolation enforcement: cannot bind two devices in the same iommu_group to vfio-pci while another is bound to a host driver. (covers REQ-O2 — group-binding constraint)
- [ ] AC-O12: vfio-pci hot-reset: `VFIO_DEVICE_PCI_HOT_RESET` resets all devices in the affected reset group atomically. (covers REQ-O1)

## Verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/vfio/eventfd_irq.tla` | `drivers/vfio/virqfd.md` (proves: hardware IRQ → virqfd kernel handler → eventfd_signal → KVM_IRQFD → vCPU IPI delivery; under concurrent IRQ + IRQ-disable + eventfd close, no signal lost or delivered after close, no double-fire) |
| `models/vfio/group_container.tla` | `drivers/vfio/group.md` (proves: legacy group-attach-to-container + simultaneous device-open + container-set-iommu sequence — concurrent open of same group from N processes serialized correctly; group-detach blocked while devices open) |
| `models/vfio/devfd_lifecycle.tla` | `drivers/vfio/device-cdev.md` (proves: device-fd open → ioctl-bind-iommufd → mmap-BAR → close-while-DMA-in-flight: every in-flight DMA either completes or is faulted via IOMMU teardown; no UAF of vfio_device) |
| `models/vfio/migration_state.tla` | `drivers/vfio/pci-mlx5-vmig.md` (proves: live-migration state machine — RUNNING → PRE_COPY → STOP_COPY → STOP → RESUMING → RUNNING; concurrent VM userspace + variant-driver state-fetch never produce a state read that violates monotonic transition order) |
| `models/vfio/msix_mmap.tla` | `drivers/vfio/pci-rdwr.md` (proves: MSI-X table mmap pass-through requires IOMMU-validated MSI-X mappable cap; without cap, mmap rejected; with cap, table-window mmap'd region is in IOMMU-reserved-region and DMA writes via this window translated correctly) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/vfio/vfio-core.md` | vfio_device refcount + close-during-bind safety: bind-iommufd holds ref; close while bound triggers ordered detach |
| `drivers/vfio/pci-rdwr.md` | `vfio_pci_rdwr_io` pre/postcondition: offset bounds-checked against region size; offset-into-config-space < 4096; offset-into-BAR < bar_size; partial-read at end of region returns short |
| `drivers/vfio/pci-config.md` | config-space write filter: every write goes through per-bit virt/RO/passthrough table; no path bypasses table; vendor-specific cap writes mediated |

## Hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-vfio_device + per-vfio_group + per-vfio_container + per-vfio_iommu refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-bus `vfio_device_ops`, `vfio_iommu_driver_ops_*`, per-mdev `mdev_driver` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | region offset + size + BAR-mmap arithmetic uses checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-bus vfio_device_ops vtables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed vfio_device + per-IRQ ctx + per-region state cleared (carries device-private state + bound iommufd ptr) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | VFIO has no JIT — N/A | § Mandatory |

### Row-2 (LSM-stackable) features for VFIO

LSM hooks called: file-LSM hooks on `/dev/vfio/*` open + ioctl + mmap + read/write. `security_vfio_*` hooks (proposed for Rookery — upstream has minimal coverage).

GR-RBAC adds:
- Per-role disallow `/dev/vfio/*` open entirely (denies passthrough capability).
- Per-role disallow NOIOMMU mode unconditionally (no role can enable unsafe DMA).
- Per-role disallow `VFIO_DEVICE_PCI_HOT_RESET` (denies cross-device reset which can affect siblings in the reset group).
- Per-role disallow live-migration (denies migration-state ioctls that vendor variants use).
- Per-role audit of every `VFIO_GROUP_GET_DEVICE_FD` + `VFIO_DEVICE_BIND_IOMMUFD` (which devices got handed to userspace, by which process).

### VFIO-specific reinforcement

- **NOIOMMU default-off + audit-logged** in Rookery (upstream is opt-in via module param; Rookery additionally logs every NOIOMMU container creation at AUDIT_USER_AVC).
- **MSI-X table-mmap protection**: VFIO_REGION_INFO_CAP_MSIX_MAPPABLE only set when IOMMU has IRQ remap + MSI-X table window covered by reserved-region; otherwise mmap of MSI-X table BAR pages denied (defense against userspace writing to MSI-X entries to redirect interrupts to arbitrary destinations).
- **vfio-pci config-space sanitization** on every read AND write (not just write): on read, hidden bits are zeroed out; on write, RO bits are silently dropped + vendor-cap writes go through per-cap filter table.
- **D3hot/D3cold transitions**: vfio-pci writes to PMCSR carefully serialized vs guest expectations; LOW_POWER_ENTRY/EXIT FEATURE multiplexer used by qemu integrates with PCI-PM core.
- **vfio_iommu_type1 + iommufd fault-isolation**: PCI-bus-error from passed-through device (e.g., AER fatal) does NOT panic the host; AER + DPC handle the fault, vfio reports VFIO_IRQ_INFO_REQ-style notification to userspace via REQ eventfd.
- **Reset-group enforcement**: `VFIO_DEVICE_PCI_HOT_RESET` requires owning fd for every device in the hot-reset group; cannot reset devices owned by other processes.
- **DMA-buf export gate**: BAR-as-dmabuf export requires CAP_SYS_RAWIO + same iommufd binding (don't allow DMA-buf to be imported into another VFIO context where IOMMU isolation differs).
- **Variant-driver migration validation**: every vendor variant driver implementing migration runs incoming RESUMING-state bytes through validation before applying (defense against malicious migration source crafting bytes that exploit destination-side variant driver bugs).
- **`/dev/vfio/vfio` (legacy container) deprecation path**: still supported for v0 but new code paths default to iommufd-cdev; legacy path scheduled for deprecation at v1 + 1 year; Rookery emits a one-shot dmesg note when legacy container is opened.

## Open Questions

(none at Tier-2 — defer to per-Tier-3 once Phase D begins; expected hot questions: legacy container deprecation timeline, NOIOMMU default policy on edge / embedded x86_64 SKUs without IOMMU at all, mdev framework future given iommufd consolidation, dmabuf-from-BAR LSM mediation semantics, default behavior for IRQ delivery when KVM_IRQFD is closed mid-IRQ)

## Out of Scope

- Per-Tier-3 (Phase D)
- vfio-platform (`platform/`), vfio-fsl-mc (`fsl-mc/`), vfio-cdx (`cdx/`) — out of v0 (ARM-only buses)
- vfio_iommu_spapr_tce (PowerPC sPAPR TCE) — out of v0
- s390-specific zPCI features (`pci/vfio_pci_zdev.c` + `pci/ism/`) — out of v0
- 32-bit-only paths
- Implementation code
