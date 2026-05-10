# Tier-2: drivers/iommu — IOMMU subsystem (Intel VT-d + AMD-Vi + iommufd + IOVA + io-pgtable + SVA + DMA-IOMMU)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/iommu/
  - include/linux/iommu.h
  - include/linux/iommufd.h
  - include/uapi/linux/iommufd.h
  - include/linux/iova.h
  - include/linux/io-pgtable.h
-->

## Summary

Tier-2 overview for the IOMMU stack: per-vendor DMA-remapping + interrupt-remapping engine drivers (Intel VT-d, AMD-Vi), the generic IOMMU API used by every DMA-capable subsystem, the iommufd UAPI replacing legacy VFIO type1 container, the IOVA allocator, the io-pgtable backends used by ARM-SMMU and DART/etc., the SVA (Shared Virtual Addressing) bridge for PCIe-PASID consumers, the irq-remapping subsystem (Intel IR + AMD IR), and the dma-iommu glue used by the kernel's `dma_*` API to mint IOVAs through an IOMMU-backed `default_domain`.

Sits between PCI/platform devices (which produce DMA requests) and the page-mapping engine that translates DMA-virtual → physical. Tightly paired with `drivers/pci/00-overview.md` ATS/PRI/PASID surface, `drivers/vfio/00-overview.md` (DMA-passthrough endpoint), `virt/kvm/00-overview.md` (interrupt-remapping integration with posted-interrupt + AVIC), `arch/x86/cpu-init.md` (Intel/AMD CPUID feature gates), `arch/x86/idt.md` (irq_remapping_*ops vector remap), `mm/00-overview.md` (page-fault handling for SVA on shared address spaces), `kernel/dma/` (separate Tier-2 future — `dma-iommu.c` is its IOMMU-backed implementation).

## Components

- **iommu core** (`iommu.c`): generic IOMMU API — `struct iommu_ops` per-vendor vtable, `struct iommu_domain` (DMA / unmanaged / identity / sva / blocked / nested types), `struct iommu_group`, device attach/detach, default-domain selection, sysfs `/sys/kernel/iommu_groups/`
- **iommu-sva** (`iommu-sva.c`): Shared Virtual Addressing — bind a process `mm_struct` to a device PASID so device DMA uses CPU page tables; consumes mmu-notifier (cross-ref `mm/00-overview.md`) for invalidation
- **io-pgfault** (`io-pgfault.c`): handle page faults from PRI-capable devices (PCIe Page Request Interface)
- **iova** (`iova.c`): generic IOVA allocator — rb-tree + per-cpu magazines (rcache) for fast alloc; consumed by dma-iommu and vfio-iommu-type1
- **iommu-pages** (`iommu-pages.c`): per-cpu page-table-page allocator with debug accounting + KMSAN integration
- **iommu-sysfs** (`iommu-sysfs.c`): per-device + per-group sysfs surface
- **iommu-traces** (`iommu-traces.c`): tracepoints (`iommu_*`)
- **iommu-debugfs** (`iommu-debugfs.c`): `/sys/kernel/debug/iommu/`
- **iommu-debug-pagealloc** (`iommu-debug-pagealloc.c`): debug page-alloc poisoning for IOMMU-backed allocations
- **dma-iommu** (`dma-iommu.c`): the kernel-DMA-API back-end that uses an IOMMU `default_domain` for `dma_alloc_coherent` / `dma_map_*`; alternative to swiotlb-direct
- **of_iommu** (`of_iommu.c`): device-tree `iommus =` property parsing
- **irq_remapping** (`irq_remapping.c` + `irq_remapping.h`): generic interrupt-remapping subsystem with per-vendor backends (Intel IRQ remap + AMD IRQ remap + Hyper-V)
- **iommufd** (`iommufd/`): the new userspace UAPI — `/dev/iommu` chardev with `IOMMU_*` ioctls (replaces VFIO type1 container); supports nested IOMMU translation (L1 page tables managed by guest, L2 by host)
- **Intel VT-d** (`intel/`): Intel IOMMU driver — DMAR ACPI table parsing (`dmar.c`), per-IOMMU-unit driver (`iommu.c`), Intel-specific PASID + PRQ handling (`pasid.c` + `prq.c`), Intel SVM (`svm.c`), nested mode (`nested.c`), per-PMU `perfmon.c`, IR (`irq_remapping.c`), debugfs, traces, cache.c (descriptor-cache invalidation)
- **AMD-Vi** (`amd/`): AMD IOMMU driver — `init.c` (ACPI IVRS table parsing + IOMMU init), `iommu.c` (per-device attach + page-table mgmt), `pasid.c`, `ppr.c` (Peripheral Page Request handling, AMD's PRI), `nested.c`, `iommufd.c` (iommufd backend integration), `quirks.c`
- **io-pgtable** (`io-pgtable.c`): generic IO page-table abstraction; backends for ARM LPAE v7s, ARM LPAE 64-bit, DART (Apple), generic-pt
- **virtio-iommu** (`virtio-iommu.c`): paravirt IOMMU driver consumed by guest VMs
- **hyperv-iommu** (`hyperv-iommu.c`): Hyper-V Synthetic Interrupt Controller integration as IRQ-remap source
- **s390-iommu** (`s390-iommu.c`): s390 zPCI IOMMU (out of v0 — keep as no-build-on-x86_64 stub)
- **Per-SoC drivers**: apple-dart, exynos, fsl_pamu, ipmmu-vmsa, msm_iommu, mtk_iommu, omap-iommu, rockchip-iommu, sprd-iommu, sun50i-iommu, tegra-smmu (out of v0; ARM-only)
- **ARM-SMMU**: `arm/{arm-smmu/, arm-smmu-v3/}` (out of v0; ARM-only)

## Scope

This Tier-2 governs `/home/doll/linux-src/drivers/iommu/` (~30 top-level files + `intel/`, `amd/`, `iommufd/`, `arm/`, `riscv/`, `generic_pt/` subdirs), public headers `include/linux/{iommu, iommufd, iova, io-pgtable, intel-iommu, amd-iommu, dma-iommu}.h`, UAPI `include/uapi/linux/iommufd.h`. ARM/RISC-V/SoC drivers compile-gated out for v0; only Intel + AMD + iommufd + virtio-iommu + hyperv-iommu in active maintenance.

## Compatibility contract — outline

### `/dev/iommu` ioctl wire format

iommufd is the modern UAPI replacing the legacy VFIO type1 container. ioctl numbers + struct layouts byte-identical to upstream UAPI (qemu / cloud-hypervisor / dpdk / spdk run unmodified).

Key categories:
- `IOMMU_DESTROY` (generic object destroy)
- `IOMMU_IOAS_ALLOC` / `IOMMU_IOAS_IOVA_RANGES` / `IOMMU_IOAS_ALLOW_IOVAS` / `IOMMU_IOAS_MAP` / `IOMMU_IOAS_COPY` / `IOMMU_IOAS_UNMAP` / `IOMMU_IOAS_MAP_FILE` / `IOMMU_IOAS_CHANGE_PROCESS`
- `IOMMU_GET_HW_INFO` (per-device IOMMU capabilities)
- `IOMMU_HWPT_ALLOC` (allocate hardware page table; supports DATA_NONE / DATA_VTD_S1 / DATA_ARM_SMMUV3 / DATA_AMD_V2 etc.)
- `IOMMU_HWPT_INVALIDATE` (selective TLB invalidate for nested mode)
- `IOMMU_HWPT_GET_DIRTY_BITMAP` / `IOMMU_HWPT_SET_DIRTY_TRACKING` (dirty-page tracking for live migration)
- `IOMMU_VFIO_IOAS` (compatibility shim for VFIO group→iommufd transition)
- `IOMMU_OPTION` (system-wide option setting)
- `IOMMU_VIOMMU_ALLOC` (virtual IOMMU object for nested guest)
- `IOMMU_VDEVICE_ALLOC` (virtual device representation)
- `IOMMU_IOAS_MAP_DMABUF` (map dmabuf into IOAS)
- `IOMMU_FAULT_QUEUE_ALLOC` (PRI fault notification eventq)

Wire-format byte-identical including alignment + reserved bytes.

### Legacy VFIO type1 container compatibility

`drivers/vfio/vfio_iommu_type1.c` consumes `struct iommu_domain` directly. iommufd provides a vfio_compat shim (`iommufd/vfio_compat.c`) so VFIO_IOMMU_GET_INFO / VFIO_IOMMU_MAP_DMA / VFIO_IOMMU_UNMAP_DMA work on a `/dev/iommu` fd masquerading as a vfio container fd. Consumer-side ABI byte-identical (cross-ref `drivers/vfio/00-overview.md`).

### `/sys/kernel/iommu_groups/`

Per-iommu-group directory:
- `<group>/devices/<dev>` — symlinks to grouped devices (DMA-aliasing siblings + ACS-disabled siblings)
- `<group>/name` — group name
- `<group>/type` — group type (DMA / DMA-FQ / identity / unmanaged / blocked)
- `<group>/reserved_regions` — IOMMU reserved IOVA ranges (MSI-X table window, etc.)

Layout + content byte-identical so `dpdk-devbind.py` / qemu's vfio-group-walk works unchanged.

### `/sys/class/iommu/`

Per-IOMMU-unit dir (e.g., `dmar0`, `dmar1`, `ivhd0`):
- `intel-iommu/{address, cap, ecap, sgaw, lp_supported, fl_supported, dl_supported, drhd_off, hawf, slts}` (Intel)
- `amd-iommu/{address, cap_off, devid, mmio_phys_end, mmio_phys_start, ...}` (AMD)
- `devices/<dev>` — symlinks to attached devices

Byte-identical for iommu-aware tooling (`dmidecode`, intel/amd-iommu-debug-tools).

### `/sys/bus/pci/devices/.../iommu_group` symlink

Each PCI device with IOMMU coverage has an `iommu_group` symlink pointing into `/sys/kernel/iommu_groups/<n>/`. Consumed by `lspci -vvv` and qemu vfio-pci probe. Identical.

### Per-device sysfs

`/sys/bus/pci/devices/.../iommu` symlink to the IOMMU unit covering this device. Identical.

### `/proc/cmdline` IOMMU options

Must parse + behave identically:
- `intel_iommu=on/off/strict/sm_on/sm_off/igfx_off/nobounce`
- `amd_iommu=on/off/fullflush/force_isolation/force_enable/force_disable`
- `amd_iommu_intr=legacy/vapic/vapic_force`
- `iommu=on/off/force/noforce/strict/passthrough/nopt/group_mf/nobypass`
- `iommu.passthrough=0/1`, `iommu.strict=0/1`, `iommu.forcedac=0/1`
- `iommu.dma_strict=0/1`
- `intremap=on/off/no_x2apic_optout/nosid`

### `/sys/kernel/debug/iommu/`

Per-domain page-table dump + per-IOMMU-unit register dump. CONFIG_IOMMU_DEBUGFS=y. Layout byte-identical for in-tree debug tools.

### Tracepoints

`/sys/kernel/tracing/events/iommu/` byte-identical names + format:
- `add_device_to_group`, `remove_device_from_group`
- `attach_device_to_domain`, `detach_device_from_domain`
- `map`, `unmap`, `map_sg`
- `io_page_fault`
- Intel-specific: `qi_submit`, `prq_report`
- AMD-specific: `amd_iommu_*`

### `dma_alloc_coherent` / `dma_map_*` glue

When IOMMU is enabled, `dma_iommu_*ops` is used as the kernel's DMA API backend. Allocates from per-device IOVA pool via `iova.c`; maps via `iommu_map`. Per-device behavior identical so unmodified drivers' DMA paths work.

### MSI-X table reserved-region

Each IOMMU group reserves the host-side MSI-X message-address window (0xFEE00000-0xFEEFFFFF on x86) so device-emitted MSI writes pass through to LAPIC unaltered (otherwise IRQ remapping would break). Visible in `reserved_regions` sysfs.

### IRQ remapping

When CONFIG_IRQ_REMAP=y + supported HW, all device-generated MSI/MSI-X messages MUST be steered through the IOMMU's IRQ-remap table. `intremap=` cmdline knob honored. `/sys/devices/system/clocksource/...` unaffected; per-device IRQ allocation continues to use Linux IRQ numbers, but the underlying vector is remapped via IRTE. Required for x2APIC + posted interrupts (KVM AVIC + KVM posted-intr).

### SVA

`iommu_sva_bind_device(dev, mm)` returns `struct iommu_sva *` representing PASID-bound mm. Consumed by Intel-IDXD, Intel-DSA, AMD-vIOMMU-PRI consumers, FPGA accelerators. Lifetime tied to mm + sva-handle refcount. Behavior identical so unmodified accelerator drivers work.

## Tier-3 docs governed by this Tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/iommu/iommu-core.md` | `iommu.c` + `iommu-priv.h`: generic IOMMU API + ops + group + domain lifecycle + default-domain selection |
| `drivers/iommu/iommu-sva.md` | `iommu-sva.c` + `io-pgfault.c`: shared virtual addressing + PRI fault handling |
| `drivers/iommu/iommu-sysfs.md` | `iommu-sysfs.c`: `/sys/kernel/iommu_groups/` + `/sys/class/iommu/` |
| `drivers/iommu/iommu-debugfs.md` | `iommu-debugfs.c`: `/sys/kernel/debug/iommu/` |
| `drivers/iommu/iommu-pages.md` | `iommu-pages.c`: per-cpu page-table-page allocator with KMSAN |
| `drivers/iommu/iova.md` | `iova.c`: IOVA allocator + per-cpu magazines (rcache) |
| `drivers/iommu/dma-iommu.md` | `dma-iommu.c`: DMA-API backend over IOMMU |
| `drivers/iommu/irq-remapping.md` | `irq_remapping.c`: generic IRQ-remap dispatch |
| `drivers/iommu/iommufd-main.md` | `iommufd/main.c` + `driver.c` + `device.c`: `/dev/iommu` chardev + ioctl dispatch + object table |
| `drivers/iommu/iommufd-ioas.md` | `iommufd/ioas.c` + `io_pagetable.c` + `pages.c` + `iova_bitmap.c`: IOAS abstraction + page mapping |
| `drivers/iommu/iommufd-hwpt.md` | `iommufd/hw_pagetable.c`: hardware page table object (paging + nested) |
| `drivers/iommu/iommufd-eventq.md` | `iommufd/eventq.c`: PRI fault eventq + delivery to userspace |
| `drivers/iommu/iommufd-viommu.md` | `iommufd/viommu.c`: virtual IOMMU object for nested guest |
| `drivers/iommu/iommufd-vfio-compat.md` | `iommufd/vfio_compat.c`: legacy VFIO type1 container shim |
| `drivers/iommu/intel-dmar.md` | `intel/dmar.c`: ACPI DMAR parsing + IOMMU unit discovery |
| `drivers/iommu/intel-iommu.md` | `intel/iommu.c` + `iommu.h`: Intel VT-d driver core + cache.c invalidation |
| `drivers/iommu/intel-pasid.md` | `intel/pasid.c` + `pasid.h` + `prq.c`: Intel PASID + PRQ |
| `drivers/iommu/intel-svm.md` | `intel/svm.c`: Intel SVM (PASID-backed shared address space) |
| `drivers/iommu/intel-nested.md` | `intel/nested.c`: Intel nested-translation (L1 guest pgtable + L2 host pgtable) |
| `drivers/iommu/intel-perfmon.md` | `intel/perfmon.c` + `perfmon.h` + `perf.c` + `perf.h`: Intel IOMMU PMU |
| `drivers/iommu/intel-irq-remap.md` | `intel/irq_remapping.c`: Intel IRQ-remap (IR) backend |
| `drivers/iommu/amd-init.md` | `amd/init.c` + `amd_iommu_types.h`: ACPI IVRS parsing + AMD IOMMU init |
| `drivers/iommu/amd-iommu.md` | `amd/iommu.c` + `amd_iommu.h`: AMD-Vi driver core |
| `drivers/iommu/amd-pasid.md` | `amd/pasid.c` + `ppr.c`: AMD PASID + Peripheral Page Request |
| `drivers/iommu/amd-nested.md` | `amd/nested.c`: AMD nested-translation |
| `drivers/iommu/amd-iommufd.md` | `amd/iommufd.c`: AMD-Vi iommufd backend integration |
| `drivers/iommu/virtio-iommu.md` | `virtio-iommu.c`: paravirt IOMMU driver |
| `drivers/iommu/hyperv-iommu.md` | `hyperv-iommu.c`: Hyper-V SynIC IRQ-remap |

## Compatibility outline (top-level)

- REQ-O1: `/dev/iommu` chardev + `IOMMU_*` ioctl wire format byte-identical to upstream UAPI (qemu / cloud-hypervisor / dpdk / spdk run unmodified).
- REQ-O2: Legacy VFIO type1 container compatibility (vfio_compat shim) byte-identical (existing qemu vfio-group code paths still work).
- REQ-O3: `/sys/kernel/iommu_groups/` + `/sys/class/iommu/` + per-device `iommu_group` symlink byte-identical.
- REQ-O4: All `intel_iommu=`, `amd_iommu=`, `iommu=`, `intremap=` cmdline options parsed identically.
- REQ-O5: Tracepoints under `events/iommu/` byte-identical names + format.
- REQ-O6: `/sys/kernel/debug/iommu/` debug surface byte-identical (CONFIG_IOMMU_DEBUGFS=y).
- REQ-O7: `dma-iommu` DMA-API backend transparent to in-tree drivers (no behavior diff vs swiotlb-direct except faster).
- REQ-O8: IRQ remapping required for x2APIC + posted interrupts; `intremap=off` honored as fallback.
- REQ-O9: SVA `iommu_sva_bind_device` API identical for in-tree consumers (Intel IDXD, FPGA SVA users).
- REQ-O10: Reserved-region MSI-X window sysfs surface byte-identical.
- REQ-O11: TLA+ models declared at this Tier-2 (PASID lifecycle, IOVA allocator concurrency, descriptor-queue submit/completion, irq-remap-table consistency, nested invalidation propagation).
- REQ-O12: Verus/Creusot Layer-4 functional contracts on iommu_map/unmap (every mapped IOVA is reachable; every unmapped IOVA produces fault on subsequent device access; no dangling page-table page).
- REQ-O13: Hardening: row-1 features applied per `00-security-principles.md`, with IOMMU-specific reinforcement (default `iommu=strict` + dma-iommu strict-flush, ACS-isolation enforcement, IRTE bounds checking).

## Acceptance Criteria (top-level)

- [ ] AC-O1: qemu `-device vfio-pci,host=...,iommufd=on` works on a passthrough NIC; guest sends/receives traffic. (covers REQ-O1)
- [ ] AC-O2: qemu `-object iommu-group,id=g0,path=/dev/vfio/N` (legacy mode) still works via vfio_compat shim. (covers REQ-O2)
- [ ] AC-O3: `find /sys/kernel/iommu_groups/ -ls` produces output matching upstream baseline structure. (covers REQ-O3)
- [ ] AC-O4: `intel_iommu=on,sm_on iommu=pt` boots correctly; `dmesg | grep DMAR` shows IOMMU init success. (covers REQ-O4)
- [ ] AC-O5: `perf record -e iommu:*` captures iommu tracepoints. (covers REQ-O5)
- [ ] AC-O6: SPDK userspace driver against vfio-pci performs DMA at same throughput as upstream baseline. (covers REQ-O7)
- [ ] AC-O7: x2APIC + KVM posted-interrupt mode active when `intremap=on` (verify via `dmesg | grep -i 'IRQ remapping'` + `cat /sys/devices/.../irq_remap_enabled`). (covers REQ-O8)
- [ ] AC-O8: idxd-cdev test program (Intel DSA) successfully binds PASID via SVA + submits work descriptor. (covers REQ-O9)
- [ ] AC-O9: kselftest `tools/testing/selftests/iommu/` passes. (covers REQ-O11, REQ-O12)
- [ ] AC-O10: Nested-mode test: qemu launches L1 guest with `-device intel-iommu` + L1 boots a Linux guest L2 with assigned PCI device — the assigned device works in L2. (covers REQ-O1, REQ-O11)
- [ ] AC-O11: dirty-page tracking: VM live-migration with iommufd HWPT_GET_DIRTY_BITMAP shows correct dirty bits during DMA. (covers REQ-O1)
- [ ] AC-O12: ACS-isolation test: two PCI devices in the same group cannot bind to different drivers — group binding enforced. (covers REQ-O3, REQ-O13)

## Verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/iommu/iova_alloc.tla` | `drivers/iommu/iova.md` (proves: IOVA allocator + per-cpu rcache + global rb-tree concurrency: alloc + free under high contention never produce duplicate IOVA, never leak slot) |
| `models/iommu/qi_descriptor.tla` | `drivers/iommu/intel-iommu.md` (proves: Intel descriptor-queue submit + completion + tail-fence + invalidation-wait observe ordering needed by spec; concurrent submit from N CPUs serializes correctly via head pointer) |
| `models/iommu/pasid_lifecycle.tla` | `drivers/iommu/iommu-sva.md` (proves: PASID alloc → bind → mm-notifier-invalidate → unbind → free state machine; mm exit + concurrent device DMA never produce use-after-free of pgtable; mmu-notifier range-invalidate flushed before underlying page reuse) |
| `models/iommu/irte_consistency.tla` | `drivers/iommu/intel-irq-remap.md` (proves: IRTE table modification + invalidation + IPI delivery: target CPU never receives stale-IRTE-redirected interrupt after remap update) |
| `models/iommu/nested_invalidate.tla` | `drivers/iommu/iommufd-hwpt.md` (proves: nested HWPT — guest-managed L1 pgtable + host-managed L2 pgtable + IOMMU_HWPT_INVALIDATE propagation: every guest-issued invalidate eventually reaches IOMMU TLB before bound device's next DMA) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/iommu/iommu-core.md` | `iommu_map(domain, iova, paddr, size, prot)` post-condition: all `[iova, iova+size)` reachable via translate; `iommu_unmap` post: no pte present in range, page-table-page leak count == 0 |
| `drivers/iommu/iova.md` | `alloc_iova` invariant: returned IOVA is within [start_pfn, limit_pfn] and not already in tree; `free_iova` invariant: never frees an unallocated IOVA |
| `drivers/iommu/iommufd-ioas.md` | `iommufd_ioas_map(ioas, iova, len, ptr)` invariant: refcount on backing pages held until unmap; concurrent map+unmap+copy on same IOAS serialized |

## Hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-domain + per-group + per-iommufd-object refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-vendor `iommu_ops`, `iommu_dirty_ops`, `iommu_domain_ops` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | IOVA arithmetic + page-table walks + descriptor-queue indices use checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-vendor iommu_ops vtables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed page-table pages cleared before return to allocator (carry stale guest mappings) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | IOMMU has no JIT — N/A surface, but page-table pages never RWX | § Mandatory |

### Row-2 (LSM-stackable) features for IOMMU

LSM hooks called: `security_iommu_*` (proposed for Rookery — upstream has minimal LSM coverage of iommu surface). For now, file-LSM hooks on `/dev/iommu` open + ioctl + `/sys/kernel/iommu_groups/` writes.

GR-RBAC adds:
- Per-role disallow `/dev/iommu` open (denies user-driver ability to set up DMA at all).
- Per-role disallow `IOMMU_HWPT_ALLOC` with nested type (denies guest-managed pgtable creation).
- Per-role disallow IRQ remap reconfiguration via debugfs.
- Per-role audit of all IOAS map operations (IOVA + paddr + size logged).

### IOMMU-specific reinforcement

- **Default `iommu=strict`** in Rookery (upstream default since 5.16; Rookery preserves the strict-flush invariant — every IOVA unmap is followed by a TLB flush before the IOVA is reused).
- **ACS isolation enforcement**: groups respect ACS PCI-bridge separation; `pcie_acs_override=` cmdline disabled by default (upstream has it as a debug-only override; Rookery makes it root+CAP_SYS_ADMIN required).
- **Reserved-region MSI-X window**: cannot be unmapped from any domain; iommu_map onto the [0xFEE00000, 0xFEEFFFFF] range rejected with -EINVAL.
- **PASID range bounds**: per-device PASID alloc gated to within IOMMU-advertised max-PASID (defense against driver passing out-of-range PASID, which on some HW corrupts adjacent PASID entries).
- **PRI fault rate-limit**: per-device PRI fault flood rate-limited; persistent overflow auto-disables PRI on the device + logs CRIT.
- **iommufd object table bounds**: per-`/dev/iommu` fd object count capped (default 64K); prevents memory exhaustion attack.
- **Nested HWPT validation**: every guest-supplied page-table root + format flag validated against IOMMU-advertised supported nested types before HWPT alloc; malformed roots rejected pre-bind (don't trust guest to follow spec).
- **dma-iommu deferred-flush window**: bounded by both time + count to avoid host RAM exhaustion under iova-flood scenarios.

## Open Questions

(none at Tier-2 — defer to per-Tier-3 once Phase D begins; expected hot questions: default `iommu.passthrough` for trusted-host vs untrusted-host scenarios, default IRQ-remap mode on x2APIC-capable but not IRQ-remap-capable HW, PASID max default per-device cap, nested-iommu enable default for KVM nested guests)

## Out of Scope

- Per-Tier-3 (Phase D)
- ARM-SMMU + SoC IOMMU drivers (`arm/`, `apple-dart.c`, `exynos-iommu.c`, `fsl_pamu*.c`, `ipmmu-vmsa.c`, `msm_iommu.c`, `mtk_iommu*.c`, `omap-iommu*.c`, `rockchip-iommu.c`, `sprd-iommu.c`, `sun50i-iommu.c`, `tegra-smmu.c`) — out of v0 since x86_64-only
- s390-iommu (out of v0)
- RISC-V IOMMU (`riscv/`) — out of v0
- io-pgtable backends used only by ARM-SMMU + DART (out of v0)
- generic_pt (only consumed by ARM/RISC-V) — out of v0
- 32-bit-only paths
- Implementation code
