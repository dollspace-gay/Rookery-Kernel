---
title: "Tier-3: drivers/vfio/pci/{vfio_pci,vfio_pci_core}.c — vfio-pci core (per-device fd lifecycle + region/IRQ exposure + reset)"
tags: ["tier-3", "drivers-vfio", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The vfio-pci core driver — the most-used VFIO bus driver, binds to PCIe devices via `driver_override = vfio-pci` (from `/sys/bus/pci/devices/.../driver_override`). Exposes a per-device file descriptor via `/dev/vfio/devices/vfioN` (modern iommufd-cdev path) or `/dev/vfio/<group>` (legacy group-fd path) that userspace VMM (qemu, cloud-hypervisor, firecracker, dpdk, spdk) drives directly. Owns: per-device fd allocation + close-during-DMA safety, BAR region exposure (mmap'd raw + emulated config-space + emulated INTx/MSI/MSI-X tables), IRQ delivery via virqfd (cross-ref `drivers/vfio/virqfd.md`), per-vendor variant-driver hookup for live-migration extensions (mlx5/pds/qat/hisilicon/nvgrace/xe/virtio).

This Tier-3 covers `vfio_pci.c` (~290 lines: pci_driver registration + binding) + `vfio_pci_core.c` (~2600 lines: per-device fd ops + region exposure + IRQ + reset).

### Acceptance Criteria

- [ ] AC-1: qemu `-device vfio-pci,host=0000:01:00.0` boots Linux guest with passthrough NVMe; guest sees + drives device.
- [ ] AC-2: cloud-hypervisor with iommufd-mode vfio-pci passthrough works.
- [ ] AC-3: SPDK NVMe userspace driver against vfio-pci-bound NVMe achieves IOPS within 2% upstream.
- [ ] AC-4: DPDK testpmd against vfio-pci-bound NIC achieves line-rate.
- [ ] AC-5: BAR mmap test: mmap'd BAR addr writes go directly to device; readback reflects HW state.
- [ ] AC-6: MSI-X test: 64-vector MSI-X allocation; per-vector eventfd registered; HW IRQ delivers as eventfd write.
- [ ] AC-7: Hot-reset test: reset-group with 2 devices owned by same fd → `VFIO_DEVICE_PCI_HOT_RESET` resets both atomically.
- [ ] AC-8: AER test: simulated AER fault → REQ eventfd kicked → qemu reports to guest.
- [ ] AC-9: kselftest `tools/testing/selftests/vfio/` passes.

### Architecture

`CoreDevice` lives in `kernel::vfio::pci::CoreDevice`:

```
struct CoreDevice {
  vdev: VfioDevice,                  // generic vfio device base (refcount, ops vtable)
  pdev: Arc<PciDev>,
  region_iter: Mutex<Vec<KBox<Region>>>,
  num_regions: u32,
  config_size: u32,
  msi_perm: Mutex<Option<MsiPermission>>,
  msix_perm: Mutex<Option<MsixPermission>>,
  bar_mask: u32,
  has_intx: AtomicBool,
  has_msi: AtomicBool,
  has_msix: AtomicBool,
  msix_bar: i8,
  msix_offset: u32,
  msix_size: u32,
  ctx: Mutex<Vec<Arc<VirqfdContext>>>,
  reset_handler: Option<fn(&CoreDevice)>,
  emu_devnums: AtomicU32,
  pm_entry: AtomicBool,
  igate: Mutex<()>,
  fault_count: AtomicU32,
  aer_state: AtomicU8,
}

struct Region {
  index: u32,
  type_: VfioRegionType,
  size: u64,
  offset: u64,
  flags: VfioRegionFlags,
  ops: Option<&'static dyn RegionOps>,
  data: KBox<dyn Any>,
}
```

PCI-driver-side `pci_driver::probe`:
1. Allocate `CoreDevice`.
2. `vfio_pci_core_init_dev(&vdev)`: assign pdev; init region table; populate per-BAR region descriptors from PCI BAR sizes.
3. `vfio_pci_core_register_device(&vdev)`: register with vfio core (creates `/sys/bus/pci/devices/.../vfio-dev/vfio0/` sysfs entry; allocates per-device chardev).

Per-device fd open path (from VFIO_GROUP_GET_DEVICE_FD or VFIO_DEVICE_BIND_IOMMUFD):
1. `vfio_pci_core_open_device(&vdev)`:
   - `vfio_pci_core_enable(&vdev)`: enable PCI device (`pci_enable_device`), save initial PCI state (config-space backup for restore on close), setup INTx mask if INTx is sole IRQ.
   - `vfio_pci_core_finish_enable(&vdev)`: setup MSI-X table emulation overlay (so VMM can virtualize); init virqfd contexts; install AER notifier.
2. Allocate fd via `anon_inode_getfd`.

Per-region read/write:
- Index by `_IOC(...)` reg_index from `pos`.
- Look up `vdev.region_iter[index]`.
- If region has `ops`: dispatch to per-region read/write callback.
- Else for BAR: use `pci_iomap` + `__raw_readl/writel` family (with per-LSM hook check on each).
- For config-space: walk per-bit virt/RO/passthrough table; emulate or pass through.

Per-region mmap:
- Validate region supports mmap (VFIO_REGION_INFO_FLAG_MMAP set).
- For BAR: `vm_iomap_memory(vma, bar_phys, bar_size)` → direct map of MMIO.
- For MSI-X table: only mappable if VFIO_REGION_INFO_CAP_MSIX_MAPPABLE set (cross-ref `drivers/vfio/00-overview.md` § Hardening — IOMMU validates).

`VFIO_DEVICE_SET_IRQS` flow:
1. Parse arg: index (INTX / MSI / MSIX / REQ / ERR), start, count, flags (DATA_EVENTFD / DATA_BOOL / DATA_NONE).
2. For each [start, start+count):
   - DATA_EVENTFD: install per-vector `VirqfdContext` linking eventfd → device IRQ.
   - DATA_BOOL: mask/unmask per-vector.
   - DATA_NONE: free per-vector ctx.

Per-IRQ delivery (kernel-side, from per-device IRQ handler):
- `request_irq(vec, virqfd_handler, ...)` installed during SET_IRQS.
- On HW IRQ: `virqfd_handler(irq, virqfd_ctx)` → `eventfd_signal(virqfd_ctx.eventfd, 1)`.
- Userspace VMM receives wakeup on poll(eventfd) OR (more commonly) KVM_IRQFD bypass kicks vCPU directly.

Reset:
- FLR: `pcie_capability_clear_and_set_word(pdev, PCI_EXP_DEVCTL, 0, PCI_EXP_DEVCTL_BCR_FLR)` → wait 100ms.
- Hot-reset: `pci_reset_function_locked(pdev)` for each device in reset-group; ownership-check (all devices must be owned by current fd).

iommufd attach: `vfio_pci_core_iommufd_bind` → calls `iommufd_device_bind(...)` from iommufd-main; `_attach_ioas` → `iommufd_device_attach(...)`. Bridges into iommufd HWPT.

### Out of Scope

- Per-vendor variant drivers (covered in `pci-mlx5-vmig.md`, `pci-pds-vmig.md`, etc. future Tier-3s)
- vfio core (covered in `drivers/vfio/vfio-core.md` future Tier-3)
- iommufd bridge details (covered in `drivers/vfio/iommufd-backend.md` future Tier-3)
- virqfd internals (covered in `drivers/vfio/virqfd.md` future Tier-3)
- vfio-pci config-space details (covered in `drivers/vfio/pci-config.md` future Tier-3)
- vfio-pci IRQ details (covered in `drivers/vfio/pci-intrs.md` future Tier-3)
- IGD opregion (covered in `drivers/vfio/pci-igd.md` future Tier-3)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vfio_pci_core_device` | per-device control block | `kernel::vfio::pci::CoreDevice` |
| `struct vfio_pci_region` | per-region descriptor | `kernel::vfio::pci::Region` |
| `vfio_pci_core_init_dev(vdev)` / `_release_dev(vdev)` | per-device construct/destroy | `CoreDevice::init` / `_release` |
| `vfio_pci_core_register_device(vdev)` / `_unregister_device(vdev)` | bind to vfio core | `CoreDevice::register` / `_unregister` |
| `vfio_pci_core_enable(vdev)` / `_disable(vdev)` | per-device open/close hooks | `CoreDevice::enable` / `_disable` |
| `vfio_pci_core_finish_enable(vdev)` | post-enable hookup (intx + msix + emulator setup) | `CoreDevice::finish_enable` |
| `vfio_pci_core_open_device(vdev)` / `_close_device(vdev)` | per-fd open/close | `CoreDevice::open_device` / `_close_device` |
| `vfio_pci_core_ioctl(vdev, cmd, arg)` | per-device fd ioctl dispatch | `CoreDevice::ioctl` |
| `vfio_pci_core_read(vdev, buf, count, ppos)` / `_write(...)` | per-region read/write | `CoreDevice::read` / `_write` |
| `vfio_pci_core_mmap(vdev, vma)` | per-region mmap | `CoreDevice::mmap` |
| `vfio_pci_core_request(vdev, count)` | request-eventfd kick | `CoreDevice::request` |
| `vfio_pci_core_match(vdev, buf)` | match userspace passed string | `CoreDevice::match` |
| `vfio_pci_core_pm_entry(vdev)` / `_exit(vdev)` | PM state transitions | `CoreDevice::pm_entry` / `_exit` |
| `vfio_pci_core_aer_err_detected(...)` | per-device AER notification | `CoreDevice::aer_err_detected` |
| `vfio_pci_core_get_irq_info(vdev, info)` | enumerate IRQ types | `CoreDevice::get_irq_info` |
| `vfio_pci_core_set_irqs(vdev, ...)` | configure IRQ delivery (eventfd binding) | `CoreDevice::set_irqs` |
| `vfio_pci_core_get_region_info(vdev, info)` / `_get_region_caps(...)` | enumerate regions | `CoreDevice::get_region_info` / `_get_region_caps` |
| `vfio_pci_core_register_dev_region(vdev, region_index, ops, data, ...)` | register custom region (e.g., IGD opregion) | `CoreDevice::register_region` |
| `vfio_pci_core_iommufd_bind(vdev, ...)` / `_unbind(...)` | iommufd binding | `CoreDevice::iommufd_bind` / `_unbind` |
| `vfio_pci_core_iommufd_attach_ioas(vdev, ioas_id)` / `_detach_ioas(vdev)` | per-IOAS attach (modern path) | `CoreDevice::iommufd_attach_ioas` / `_detach_ioas` |
| `vfio_pci_core_pci_hot_reset(vdev, ...)` | hot-reset infrastructure | `CoreDevice::pci_hot_reset` |
| `vfio_pci_set_power_state(vdev, state)` | D-state transitions | `CoreDevice::set_power_state` |

### compatibility contract

REQ-1: `pci_driver::probe(pdev)` accepts every PCIe device whose `driver_override` is set to `vfio-pci`; calls `vfio_pci_core_init_dev` + `vfio_pci_core_register_device`.

REQ-2: Per-device fd via `anon_inode_getfd("[vfio-pci]", &vfio_pci_fops, vdev, O_RDWR | O_CLOEXEC)` — mmappable, ioctl-controllable.

REQ-3: VFIO ioctls `VFIO_DEVICE_GET_INFO`, `_GET_REGION_INFO`, `_GET_IRQ_INFO`, `_SET_IRQS`, `_RESET`, `_QUERY_GFX_PLANE`, `_GET_PCI_HOT_RESET_INFO`, `_PCI_HOT_RESET`, `_IOEVENTFD`, `_FEATURE` byte-identical wire-format dispatch.

REQ-4: Per-region exposure: VFIO_PCI_BAR0_REGION_INDEX..VFIO_PCI_BAR5_REGION_INDEX (BARs 0-5), VFIO_PCI_ROM_REGION_INDEX (PCI ROM), VFIO_PCI_CONFIG_REGION_INDEX (config-space), VFIO_PCI_VGA_REGION_INDEX (VGA legacy), plus per-vendor `index = VFIO_PCI_NUM_REGIONS + N` (IGD opregion, IGD lpc-bridge, mlx5-migration, qat-migration, etc.).

REQ-5: BAR-region caps: VFIO_REGION_INFO_CAP_SPARSE_MMAP (per-region sparse mmap descriptor for partial-mappable BARs), VFIO_REGION_INFO_CAP_TYPE (per-vendor region type), VFIO_REGION_INFO_CAP_MSIX_MAPPABLE (per-spec MSI-X table mappable when IOMMU-validates).

REQ-6: BAR mmap: when caps allow, `mmap(devfd, size, PROT_READ|PROT_WRITE, MAP_SHARED, offset = region_offset)` produces direct mapping of underlying BAR MMIO. Userspace driver writes to mapped addr → goes straight to device.

REQ-7: Config-space access via read/write at `offset = region_offset + reg_addr`: per-bit virt/RO/passthrough table determines whether read returns raw value, emulated value, hidden (zeros), AND whether write goes through, drops, or virtualizes.

REQ-8: IRQ delivery via virqfd: `VFIO_DEVICE_SET_IRQS` with `VFIO_IRQ_SET_DATA_EVENTFD` registers per-vector eventfd; on hardware IRQ, kernel-side virqfd writes 1 to eventfd. qemu typically pairs each eventfd with a KVM_IRQFD for guest IRQ injection without userspace round-trip.

REQ-9: Reset: `VFIO_DEVICE_RESET` triggers per-device function-level reset (FLR) if supported, else PM-based D3hot-D0 cycle, else secondary-bus reset. Hot-reset (`VFIO_DEVICE_PCI_HOT_RESET`) atomically resets all devices in the affected reset group.

REQ-10: AER integration: per-device AER fault → `vfio_pci_core_aer_err_detected` → notification via REQ eventfd to userspace (qemu can then trigger guest-side response).

REQ-11: PM integration: per-device runtime-PM coordinated with userspace via `VFIO_DEVICE_FEATURE_LOW_POWER_ENTRY` / `_LOW_POWER_EXIT` features.

REQ-12: iommufd binding: modern path uses `vfio_pci_core_iommufd_bind` + `_attach_ioas`; legacy path uses container fd attachment. Both work transparently.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dev_no_uaf` | UAF | `Arc<CoreDevice>` outlives all per-device fds; close-during-DMA safe via per-device IGate. |
| `region_idx_no_oob` | OOB | `region_iter[index]` lookup bounds-checked. |
| `bar_offset_no_oob` | OOB | per-region read/write offset checked against region.size. |
| `irq_vec_no_oob` | OOB | per-IRQ vector index < num_msix or num_msi or 1 (INTX). |
| `pm_state_valid` | TRANSITION | D-state transitions follow PCI spec (D0 → D1 → D2 → D3hot → D0); never invalid jump. |

### Layer 2: TLA+

`models/vfio/devfd_lifecycle.tla` (parent-declared): proves device-fd open → bind-iommufd → mmap-BAR → close-while-DMA-in-flight: every in-flight DMA either completes or is faulted via IOMMU teardown; no UAF of vfio_device.
`models/vfio/eventfd_irq.tla` (parent-declared): proves hardware IRQ → virqfd kernel handler → eventfd_signal → KVM_IRQFD → vCPU IPI delivery; no signal lost.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `CoreDevice::open_device` post: PCI device enabled, config-space saved, virqfd contexts initialized, AER notifier installed | `CoreDevice::open_device` |
| `CoreDevice::close_device` post: in-flight DMA completed or aborted via IOMMU detach; PCI device disabled; config-space restored | `CoreDevice::close_device` |
| `CoreDevice::set_irqs` post: per-vector context installed iff DATA_EVENTFD; old context properly destroyed before new install | `CoreDevice::set_irqs` |
| `CoreDevice::pci_hot_reset` precondition: all devices in reset-group are owned by current fd | `CoreDevice::pci_hot_reset` |

### Layer 4: Verus/Creusot functional

`vfio-pci-bound device → qemu KVM_RUN guest → guest DMA → completes` round-trip: any DMA initiated by guest reaches host RAM at iommu-translated address, delivers MSI-X via virqfd → eventfd → KVM_IRQFD chain to guest vCPU. Encoded as Verus model: `forall vfio dev guest. dma_round_trip(vfio, dev, guest) succeeds iff iommu_attached(dev) && msix_routed(dev)`.

### hardening

(Inherits row-1 features from `drivers/vfio/00-overview.md` § Hardening.)

vfio-pci-core specific reinforcement:

- **MSI-X table-mmap protection** — VFIO_REGION_INFO_CAP_MSIX_MAPPABLE only set when IOMMU has IRQ remap + MSI-X table covered by reserved-region; otherwise mmap of MSI-X table BAR pages denied (defense against userspace driver writing MSI-X table entries to redirect interrupts to arbitrary destinations).
- **vfio-pci config-space sanitization** on every read AND write — hidden bits zeroed on read; RO bits silently dropped on write; per-cap filter applied (defense against config-space attacks via VMM bug).
- **D-state transition validation** — VFIO_DEVICE_FEATURE_LOW_POWER_ENTRY accepted only in D0; transitions follow PCI spec (defense against malformed VMM driving device into invalid PM state).
- **Reset-group ownership enforcement** — `VFIO_DEVICE_PCI_HOT_RESET` requires owning fd for every device in the group; defense against cross-tenant device-reset.
- **AER fault count rate-limit** — per-device AER fault rate-limited; over-rate triggers per-device WARN + suspend further AER notification.
- **Per-fd region count cap** — per-device region count bounded (default 64); defense against per-vendor variant driver registering excessive regions.
- **Per-IRQ context refcount** — per-vector VirqfdContext refcounted; close-during-IRQ safe.
- **PCI-bus-error from passed-through device** does NOT panic the host; AER + DPC handle the fault, vfio reports REQ eventfd notification to userspace.
- **Vendor variant-driver migration validation** — per-vendor variant driver implementing migration runs incoming RESUMING-state bytes through validation before applying (defense against malicious migration source crafting bytes that exploit destination-side variant driver bugs).

