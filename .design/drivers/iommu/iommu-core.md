# Tier-3: drivers/iommu/iommu.c — Generic IOMMU API (ops + domain + group + device-attach lifecycle + default-domain selection)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/iommu.c
  - drivers/iommu/iommu-priv.h
  - include/linux/iommu.h
-->

## Summary

The vendor-agnostic IOMMU API — every per-vendor IOMMU driver (Intel VT-d, AMD-Vi, ARM-SMMU, virtio-iommu, hyperv-iommu) implements `struct iommu_ops` and registers via `iommu_device_register`. This Tier-3 covers `drivers/iommu/iommu.c` (~4100 lines): vendor-agnostic abstractions for `struct iommu_domain` (DMA / unmanaged / identity / sva / blocked / nested types), `struct iommu_group` (set of devices that must share an address space because of DMA-aliasing or ACS-isolation gaps), per-device attach/detach, default-domain selection per per-device + per-group policy, sysfs `/sys/kernel/iommu_groups/`, IOMMU-fault dispatch, IOMMU-cgroup integration, IOMMU resv-region enumeration. Also owns the `iommu_dma_*` API entry points that bridge into `kernel/dma/dma-iommu.md` for DMA-API consumption.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iommu_ops` | per-vendor vtable (alloc_domain / probe_device / device_group / etc.) | `kernel::iommu::IommuOps` (trait) |
| `struct iommu_domain` | per-IOAS abstract domain | `kernel::iommu::Domain` |
| `struct iommu_group` | grouped devices sharing address space | `kernel::iommu::Group` |
| `iommu_device_register(iommu, ops, hw_dev)` | register vendor IOMMU instance | `IommuDevice::register` |
| `iommu_device_unregister(iommu)` | inverse | `IommuDevice::unregister` |
| `iommu_probe_device(dev)` | per-device probe (called from bus probe) | `Device::probe_iommu` |
| `iommu_release_device(dev)` | per-device release | `Device::release_iommu` |
| `iommu_group_alloc()` / `_get_for_dev` | alloc / lookup group | `Group::alloc` / `Group::get_for_device` |
| `iommu_group_add_device(group, dev)` / `_remove_device` | per-device group membership | `Group::add_device` / `_remove_device` |
| `iommu_domain_alloc(bus)` / `_alloc_paging` / `_alloc_sva` | alloc domain of given type | `Domain::alloc` |
| `iommu_domain_free(domain)` | free domain | `Domain::free` (Drop) |
| `iommu_attach_device(domain, dev)` / `_detach_device` | per-device attach | `Domain::attach_device` / `_detach_device` |
| `iommu_attach_group(domain, group)` / `_detach_group` | per-group attach | `Domain::attach_group` / `_detach_group` |
| `iommu_map(domain, iova, paddr, sz, prot, gfp)` / `_unmap` | map/unmap pages | `Domain::map_pages` / `_unmap_pages` |
| `iommu_map_sg(domain, iova, sg, nents, prot, gfp)` | map sg-list | `Domain::map_sg` |
| `iommu_iova_to_phys(domain, iova)` | translate | `Domain::iova_to_phys` |
| `iommu_get_resv_regions(dev, &list)` / `_put_resv_regions` | enumerate IOMMU-reserved IOVA ranges (MSI-X window etc.) | `Device::iommu_resv_regions` |
| `iommu_register_device_fault_handler(dev, handler, data)` / `_unregister_*` | per-device fault handler | `Device::register_fault_handler` |
| `iommu_report_device_fault(dev, evt)` | dispatch fault to registered handler | `Device::report_fault` |
| `iommu_setup_default_domain(dev)` | resolve per-device default domain (DMA / IDENTITY / DMA_FQ / etc.) | `Device::setup_default_domain` |
| `iommu_get_dma_strict()` / `_set_*` | strict-flush mode | `dma_strict()` global |
| `iommu_request_dm_for_dev(dev)` / `_request_dma_domain_for_dev` | force IDENTITY or DMA domain on dev | `Device::request_iommu_domain` |
| `iommu_sva_bind_device(dev, mm)` / `_unbind` | SVA bind (cross-ref `drivers/iommu/iommu-sva.md`) | `Device::sva_bind` / `_unbind` |

## Compatibility contract

REQ-1: Per-vendor `iommu_ops` registration via `iommu_device_register` accepted source-compat for in-tree IOMMU drivers (intel/amd/arm-smmu/virtio-iommu/hyperv-iommu).

REQ-2: Per-device `iommu_group` membership reflects DMA-alias topology + ACS isolation: devices that DMA-alias each other OR share an upstream ACS-non-isolating bridge MUST be in the same group (defense against cross-device DMA leakage).

REQ-3: `/sys/kernel/iommu_groups/<N>/{devices/, name, type, reserved_regions}` byte-identical (lspci -vvv + `dpdk-devbind.py` + qemu vfio-group walk consume).

REQ-4: Per-device `iommu_group` symlink at `/sys/bus/pci/devices/.../iommu_group` byte-identical.

REQ-5: Default-domain selection per `iommu.passthrough=` cmdline + `iommu.strict=` cmdline + per-device `IOMMU_GROUP_DMA` requirement (e.g., RMRRs from IOMMU FW table reserve identity-mapped ranges).

REQ-6: `iommu_map` / `_unmap` semantics: `iommu_map(domain, iova, paddr, size, prot, gfp)` post-condition: every IOVA in [iova, iova+size) translates to (paddr + offset) with `prot` permissions until `iommu_unmap` is called. Concurrent map+unmap on disjoint ranges allowed (per-domain RW-mutex on range tree).

REQ-7: Reserved-region enumeration includes (at minimum) MSI-X window [0xFEE00000, 0xFEEFFFFF] on x86, per-device RMRR ranges from DMAR/IVRS firmware tables, software-MSI window for ARM-GIC.

REQ-8: Per-device fault handler invoked exactly once per IOMMU-detected fault (page-fault from PRI device, translation-fault from non-PRI device, fatal fault for invalid AT/AR access).

## Acceptance Criteria

- [ ] AC-1: `find /sys/kernel/iommu_groups/ -ls` produces output structurally matching upstream baseline on reference HW (Intel + AMD).
- [ ] AC-2: vfio-pci passthrough of NVMe drive: `echo vfio-pci > driver_override` + `echo $bdf > unbind` + `echo $bdf > bind`; qemu `-device vfio-pci,host=$bdf` boots guest with NVMe accessible.
- [ ] AC-3: Reserved-region test: every MSI-X table BAR area visible via `cat /sys/kernel/iommu_groups/<N>/reserved_regions` includes [0xFEE00000, 0xFEEFFFFF].
- [ ] AC-4: ACS-isolation test: PCI bridge with ACS enabled → downstream devices in separate groups; bridge with ACS disabled → all downstream in same group.
- [ ] AC-5: `iommu.passthrough=1` cmdline boots with default-IDENTITY domain; native PCIe DMA bypasses translation.
- [ ] AC-6: kselftest `tools/testing/selftests/iommu/` passes.
- [ ] AC-7: Per-device fault handler test: misbehaving device → fault delivered to handler exactly once.

## Architecture

`Domain` lives in `kernel::iommu::Domain`:

```
struct Domain {
  refcount: Refcount,
  ops: &'static dyn IommuOps,
  type_: DomainType,    // DMA / DMA_FQ / IDENTITY / SVA / BLOCKED / NESTED / UNMANAGED
  geometry: DomainGeometry,  // valid IOVA range
  pgsize_bitmap: u64,   // supported page sizes
  vendor_priv: KBox<dyn Any>,  // vendor-specific page-table state
  iotlb_sync_map: AtomicBool,
  iova_cookie: Option<KBox<IovaDomain>>,  // for DMA / DMA_FQ types — bridge to dma-iommu
}
```

`Group` lives in `kernel::iommu::Group`:

```
struct Group {
  id: u32,
  refcount: Refcount,
  name: KString,
  type_: GroupType,
  default_domain: Mutex<Option<Arc<Domain>>>,
  blocking_domain: Mutex<Option<Arc<Domain>>>,
  attached_domain: Mutex<Option<Arc<Domain>>>,
  devices: Mutex<Vec<Arc<Device>>>,
  reserved_regions: Mutex<Vec<ResvRegion>>,
  kobj: KObject,  // /sys/kernel/iommu_groups/<id>/
}
```

Per-device probe path:
1. Bus driver finishes per-bus probe (e.g., `pci_device_add`).
2. `iommu_probe_device(dev)`:
   - Call vendor `ops->probe_device(dev)` → returns `iommu_device *iommu`.
   - `iommu_group_get_for_dev(dev)` → finds existing group OR creates new (consults `ops->device_group(dev)` for grouping policy).
   - Group::add_device(group, dev).
   - Setup default domain via `iommu_setup_default_domain(dev)`:
     - Inspect `iommu.passthrough=`, `iommu.strict=`, RMRR requirements, group preferred type.
     - Resolve to one of: DMA / DMA_FQ / IDENTITY.
     - `Domain::alloc(bus, type)`.
     - `Domain::attach_group(group)`.
3. Userspace + kernel-DMA-API consumers can now `dma_map_*` (cross-ref `kernel/dma/dma-iommu.md`) which routes through this domain.

Per-device detach (e.g., for vfio-pci binding):
1. `iommu_detach_group(default_domain, group)`.
2. vfio creates its own domain (UNMANAGED type) + `iommu_attach_group(unmanaged_domain, group)`.
3. vfio uses `iommu_map` directly for its IOAS (legacy) or via iommufd (new).

Per-domain page mapping:
- `iommu_map(domain, iova, paddr, sz, prot, gfp)`:
  1. Validate iova + sz against `domain->geometry`.
  2. Validate sz alignment against `domain->pgsize_bitmap`.
  3. Iterate over the size in supported page-size chunks.
  4. Call `ops->map_pages(domain, iova_chunk, paddr_chunk, pgsize, count, prot, gfp, &mapped)`.
  5. Accumulate mapped bytes; on partial failure, unmap previously-mapped chunks before returning error.

Per-vendor page-walk + invalidation invoked through `ops->iotlb_sync_map` / `ops->iotlb_sync` after batch of map/unmap (deferred-flush optimization for DMA_FQ mode).

Sysfs:
- `/sys/kernel/iommu_groups/<N>/devices/` — symlinks to grouped `/sys/bus/pci/devices/<bdf>` etc.
- `/sys/kernel/iommu_groups/<N>/name` — group name (vendor-specific).
- `/sys/kernel/iommu_groups/<N>/type` — current type (DMA / DMA-FQ / identity / blocked / unmanaged).
- `/sys/kernel/iommu_groups/<N>/reserved_regions` — IOMMU-reserved IOVA ranges (MSI-X, RMRR, etc.) one-per-line "0xSTART 0xEND TYPE".
- Per-device `/sys/bus/pci/devices/<bdf>/iommu_group` symlink to `/sys/kernel/iommu_groups/<N>/`.
- Per-device `/sys/bus/pci/devices/<bdf>/iommu` symlink to `/sys/class/iommu/<unit>/`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `domain_no_uaf` | UAF | `Arc<Domain>` outlives any in-flight `iommu_map` call. |
| `group_no_uaf` | UAF | `Arc<Group>` outlives any device attached to it. |
| `iommu_map_no_oob` | OOB | `iova + size` checked against `domain->geometry.aperture_end`. |
| `iommu_map_align_check` | OOB | `sz` is a multiple of one of the `pgsize_bitmap` page sizes. |

### Layer 2: TLA+

(none specific to this Tier-3 — covered by parent's `models/iommu/iova_alloc.tla`, `qi_descriptor.tla`, `pasid_lifecycle.tla`, `irte_consistency.tla`, `nested_invalidate.tla`.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Every device in a group has same `iommu_ops` (single-vendor groups) | `Group::add_device` |
| `iommu_setup_default_domain` always resolves to a non-null Domain (never silent-fail) | `Device::setup_default_domain` |
| `Domain::map` post: `Domain::iova_to_phys(iova) == paddr` for every page in the mapped range | `Domain::map_pages` |
| `Domain::unmap` post: `Domain::iova_to_phys(iova) == 0` for every page in unmapped range | `Domain::unmap_pages` |

### Layer 4: Verus/Creusot functional

`Domain::map(iova, paddr, sz, prot)` ↔ inverse `Domain::unmap(iova, sz)` model: post-`unmap`, no IOVA in the range translates; subsequent `map` of the same IOVA range with different `paddr` succeeds + reflects in `iova_to_phys`. Encoded as Verus invariant on the page-table abstract state.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

iommu-core specific reinforcement:

- **Reserved-region MSI-X window enforced** — `iommu_map` onto [0xFEE00000, 0xFEEFFFFF] returns -EINVAL. Defense against userspace driver mapping IOVA over MSI-X window which would intercept LAPIC interrupts.
- **Per-group `attached_domain` mutually-exclusive** — at most one domain attached per group at any time. Prevents cross-domain DMA via group co-attach.
- **ACS isolation enforced at group construction** — `pci_acs_enabled(dev, ACS_*)` consulted during `iommu_group_get_for_dev`; ACS-non-isolating downstream devices placed in same group as upstream.
- **Per-fault handler invoked under per-device serializing lock** — defense against concurrent fault delivery causing handler re-entry.
- **`iommu.passthrough=1` requires CAP_SYS_ADMIN** at boot — Rookery-side: cmdline parser rejects this from non-init ramdisk source if SecureBoot active.
- **Per-domain refcount saturating** — overflow saturates; never wraps.
- **Default-domain swap on `/sys/kernel/iommu_groups/<N>/type` write** mediated by LSM hook + CAP_SYS_ADMIN (defense against userspace forcing IDENTITY mode on group it doesn't own).
- **Pgsize_bitmap validated against `pgsize_bitmap` advertised by vendor** — never accept map size unsupported by HW (defense against malformed driver request).

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on every `/sys/kernel/iommu_groups/*` sysfs write (default-domain swap, reserved-region add); VFIO-mediated `IOMMUFD_*` ioctl arg-len validated.
- **PAX_KERNEXEC** — W^X across iommu ops vtables; page-table pages allocated for IO-pgtable are RW from kernel, never executable.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization at every IOMMU sysfs / iommufd / VFIO ioctl entry.
- **PAX_REFCOUNT** — saturating refcount on `iommu_group`, `iommu_domain`, `iommu_device`, per-device `iommu_fwspec`, per-domain `iommu_dma_cookie`.
- **PAX_MEMORY_SANITIZE** — zero-on-free for IO-pgtable pages (contain physaddrs of host memory; a leak after free could leak adjacent PT entries to a new domain).
- **PAX_UDEREF / PAX_MEMORY_UDEREF** — strict user-pointer access on iommufd ioctl args; userspace-supplied IOVA never blindly used as kernel pointer.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `iommu_ops`, `iommu_domain_ops`, `iommu_iotlb_gather` callbacks, per-fault `iopf_handler`, `iommu_device_ops`.
- **GRKERNSEC_IO** — driver never calls `iopl/ioperm`; IOMMU MMIO via `request_mem_region` + `ioremap` only.
- **GRKERNSEC_HIDESYM** — `/sys/kernel/iommu_groups/<N>/devices/*` kernel pointers, IOMMU-MMIO base addresses, IO-pgtable roots masked from non-CAP_SYSLOG.
- **GRKERNSEC_DMESG** — IOMMU-fault, DMAR-fault, IRQ-remapping-fault, IOPF log lines restricted to CAP_SYSLOG (faults leak guest IOVA and target host phys-addr inference).
- **GRKERNSEC_TPE** — Trusted Path Execution for `/dev/iommu` (iommufd) and `/dev/vfio/*` (VFIO group/container) — these give a userspace process direct DMA-mapping capability.
- **GRKERNSEC_KMOD** — iommu driver auto-load (intel-iommu, amd-iommu, arm-smmu, virtio-iommu) gated; CAP_SYS_MODULE required.
- **GRKERNSEC_MODHARDEN** — module signature required for `iommu`, all vendor IOMMU drivers; defense against rootkit replacing iommu_ops to disable isolation.
- **CAP_SYS_RAWIO + CAP_SYS_ADMIN** strict for `iommufd` device-bind, VFIO container-attach, default-domain type swap, reserved-region add, `iommu.passthrough=1` cmdline.
- **GRKERNSEC_BRUTE** — per-device IOMMU-fault storm rate-limited; defense against malicious VF flooding the fault queue to mask a real attack.
- **PAX-anti-DMA-from-userland** — VFIO/iommufd DMA-map pins user pages with `pin_user_pages` + accounting against `RLIMIT_MEMLOCK`; never permits mapping kernel pages into a userspace-owned domain.
- **ACS-isolation strict-mode** — Rookery refuses to bind VFIO to a device whose IOMMU group spans multiple non-isolated PCIe functions; defense against peer-DMA-leak between assigned-to-guest and host-owned devices.
- **iommu.strict=1 default** — IOTLB invalidation done synchronously on unmap; defense against use-after-unmap DMA (queued-invalidation race window).
- **SecureBoot+IOMMU coupling** — when SecureBoot is enabled, Rookery refuses to honor `iommu=off` / `intel_iommu=off` / `amd_iommu=off` cmdline; defense against firmware-level DMA-attack vector.

Per-driver rationale: the IOMMU core is the linchpin of every DMA-isolation guarantee in the system — VFIO/iommufd-assigned devices, virtio-iommu paravirt, peer-DMA between PCIe devices, and confidential-VM IO trust all depend on it; a single mis-validated map call that places an IOVA over the MSI-X window can hijack interrupts and a default-domain swap to IDENTITY mode disables isolation entirely. Grsec reinforces by gating every isolation-affecting sysfs/ioctl behind CAP_SYS_ADMIN + LSM, signing every iommu_ops vtable, sanitizing IO-pgtable pages on free, hiding fault details from non-CAP_SYSLOG (faults are an information-disclosure channel), enforcing `iommu.strict=1` synchronous invalidation, and tying IOMMU enablement to SecureBoot so a malicious bootloader cannot disable isolation at boot.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- IOMMU-SVA (covered in `drivers/iommu/iommu-sva.md` future Tier-3)
- IOVA allocator (covered in `drivers/iommu/iova.md` future Tier-3)
- DMA-API backend (covered in `kernel/dma/dma-iommu.md` future Tier-3)
- IRQ-remap (covered in `drivers/iommu/irq-remapping.md` future Tier-3)
- Per-vendor implementation (Intel `intel-iommu.md`, AMD `amd-iommu.md`, etc.)
- iommufd UAPI (covered in `drivers/iommu/iommufd-*.md` future Tier-3s)
- 32-bit-only paths
- Implementation code
