# Tier-3: drivers/iommu/intel/dmar.c — DMAR ACPI table parsing + per-IOMMU-unit discovery + RMRR enumeration

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/intel-iommu.md
upstream-paths:
  - drivers/iommu/intel/dmar.c
  - drivers/iommu/intel/perf.c
  - drivers/iommu/intel/perfmon.c
  - include/linux/dmar.h
  - drivers/iommu/intel/dmar.h
-->

## Summary

DMAR (DMA Remapping) is the ACPI table that BIOS produces to enumerate Intel VT-d hardware. `drivers/iommu/intel/dmar.c` (~2400 lines) walks the table at boot, extracts per-IOMMU-unit register-base addresses + per-unit scope (which PCI devices each IOMMU governs) + RMRR (Reserved-Memory Region Reporting) entries (devices that need identity-mapped DMA at boot for legacy USB or ME), creates `struct dmar_drhd_unit` per IOMMU + maps registers + initializes per-unit interrupt-remapping. Also parses ATSR (Address Translation Services Reporting), RHSA (Remapping Hardware Static Affinity), SATC (SoC Address Translation Cache).

Critical for: Intel-IOMMU bringup (without DMAR parse, intel-iommu cannot find any IOMMU unit), boot-time DMA-quirk detection (RMRR for USB-keyboard during BIOS-USB-handoff), per-IOMMU NUMA-affinity (RHSA), per-IOMMU performance monitoring (perf.c + perfmon.c add IOMMU PMU as perf-event source).

This Tier-3 covers `drivers/iommu/intel/dmar.c` (~2391 lines) + `perf.c` (~164 lines) + `perfmon.c` (~790 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dmar_drhd_unit` | per-IOMMU-unit | `drivers::iommu::intel::DrhdUnit` |
| `struct dmar_rmrr_unit` | per-RMRR entry | `RmrrUnit` |
| `struct dmar_atsr_unit` | per-ATSR entry | `AtsrUnit` |
| `struct dmar_satc_unit` | per-SATC entry | `SatcUnit` |
| `dmar_table_init()` | initial DMAR table parse | `DmarTable::init` |
| `dmar_dev_scope_init()` | resolve dev-scope to PCI dev pointers | `DmarTable::dev_scope_init` |
| `parse_dmar_table()` | walk DMAR table entries | `DmarTable::parse` |
| `dmar_parse_one_drhd(header, arg)` | parse one DRHD entry | `DmarTable::parse_one_drhd` |
| `dmar_parse_one_rmrr(...)` | parse one RMRR entry | `DmarTable::parse_one_rmrr` |
| `dmar_parse_one_atsr(...)` | parse one ATSR entry | `DmarTable::parse_one_atsr` |
| `dmar_parse_one_satc(...)` | parse one SATC entry | `DmarTable::parse_one_satc` |
| `dmar_walk_remapping_entries(...)` | walk remap-table entries | `DmarTable::walk_remap_entries` |
| `dmar_walk_dev_scope(...)` | walk per-unit dev-scope list | `DrhdUnit::walk_dev_scope` |
| `dmar_acpi_dev_scope_status` | per-dev-scope status (resolved/pending) | `DevScope::status` |
| `alloc_iommu(drhd)` | allocate `struct intel_iommu` per-DRHD | `DrhdUnit::alloc_iommu` |
| `intel_iommu_init()` | top-level init driver from DMAR | `DrhdUnit::iommu_init` |
| `dmar_table_print_dmar_entry(...)` | per-entry pretty-print | `DmarTable::print_entry` |
| `dmar_register_bus_notifier()` | register PCI hotplug notifier | `DmarTable::register_bus_notifier` |
| `dmar_pci_bus_notifier(...)` | per-PCI-hotplug callback | `DmarTable::pci_bus_notifier` |
| `iommu_pmu_register(iommu)` (perfmon.c) | register per-IOMMU PMU | `IommuPmu::register` |
| `iommu_pmu_unregister(iommu)` | unregister per-IOMMU PMU | `IommuPmu::unregister` |
| `pmu_get_event(...)` / `pmu_set_event(...)` | per-PMU event-counter ops | `IommuPmu::get_event` / `_set_event` |

## Compatibility contract

REQ-1: ACPI DMAR table parse byte-identical:
- Header validation: signature "DMAR", checksum, OEM-ID.
- Per-entry walk: type ID dispatches to per-type parser.
- Type 0 (DRHD): per-IOMMU register base + segment-num + flags + dev-scope list.
- Type 1 (RMRR): reserved-memory range + dev-scope list (typically USB ports + ME).
- Type 2 (ATSR): per-segment ATS support + dev-scope list (specific endpoints).
- Type 3 (RHSA): NUMA proximity domain per-IOMMU base addr.
- Type 4 (ANDD): ACPI-namespace device descriptor (used for device-scope by ACPI path).
- Type 5 (SATC): SoC address-translation cache (used for IGFX type-3 specially).

REQ-2: Per-DRHD parse:
- `struct dmar_drhd_unit`:
  - reg_base_addr (typically 0xFED9_0000+)
  - reg_size (4KiB or 8KiB depending on rev)
  - segment (PCI segment 0..N)
  - flags (INCLUDE_ALL = governs all devices in segment except those in other DRHDs)
  - dev_scope[] (per-device list: BDF + dev_type) — empty if INCLUDE_ALL.
- `alloc_iommu(drhd)`: ioremap reg_base_addr; allocate `intel_iommu`; populate cap/ecap registers from MMIO; create iommu->id.

REQ-3: Per-RMRR parse:
- Reserved memory range [base..end] that listed devices need direct DMA access to during BIOS-handoff (USB legacy, ME firmware, igfx).
- Identity-map this range in IOMMU page table for listed devices at IOMMU bringup; never let userspace VFIO re-map it.

REQ-4: Per-ATSR parse:
- Per-segment ATSR entry indicating which endpoints support PCIe ATS (Address Translation Service); used by intel-pasid + iommu-sva.

REQ-5: Per-RHSA parse:
- Per-IOMMU NUMA-proximity domain; used by per-IOMMU page-table allocator + per-IOMMU work-queue NUMA affinity.

REQ-6: Per-SATC parse:
- SoC-cache flush target list; per-CPU/per-IOMMU SoC-level cache to flush on certain operations.

REQ-7: dev-scope resolution (`dmar_dev_scope_init`): two-phase:
- Phase 1 at boot: collect dev-scope BDF + path; PCI tree may not be enumerated yet.
- Phase 2 at PCI bus probe: callback resolves BDF → pci_dev pointer; `dmar_pci_bus_notifier` watches PCI hotplug.

REQ-8: PCI hotplug awareness:
- `dmar_pci_bus_add_dev(dev)`: per-new-pci-dev call resolves which DRHD/RMRR/ATSR governs it.
- `dmar_pci_bus_del_dev(dev)`: per-removed-pci-dev call clears scope reference.
- Used to maintain per-DRHD device-list across boot-time + hotplug.

REQ-9: Per-IOMMU MMIO register init (in `alloc_iommu`):
- Read CAP register (offset 0x08): per-IOMMU capability bitmap (NW=non-write-coherence, MAMV=max-addr-mask-value, etc.).
- Read ECAP register (offset 0x10): extended cap (PT=pass-through, SC=snoop-control, EAFS=ext-AFS-support, PASID, etc.).
- Read VER register (offset 0x00): per-IOMMU version major/minor.
- Read GSTS register (offset 0x1C): global status (TES=translation-enable, etc.).

REQ-10: Per-IOMMU PMU registration (perfmon.c):
- IOMMU PMU presents itself as Linux `perf_event` source.
- Per-event: counter selectable from per-spec event list (DMA hits/misses, page-walk events, IOTLB invalidation count).
- Per-CPU events delivered via per-IOMMU MSI when counter overflow.

REQ-11: Per-IOMMU perf-stat (perf.c):
- Per-IOMMU SoC-level perf-event counters incrementing per-IOMMU-op (used by tracing).

## Acceptance Criteria

- [ ] AC-1: Boot test on Intel x86_64 host: `dmesg | grep "DMAR"` shows DMAR table parse output matching upstream pattern; per-DRHD line printed.
- [ ] AC-2: Per-IOMMU detect: Skylake/Cascade Lake / Ice Lake host with multi-DRHD shows per-segment IOMMU detect.
- [ ] AC-3: RMRR identity-map test: USB keyboard usable from boot through Linux init (no DMA-fault during BIOS-USB-handoff).
- [ ] AC-4: NUMA-IOMMU affinity: 2-socket host with 2 IOMMUs shows each IOMMU bound to per-socket NUMA node via RHSA.
- [ ] AC-5: PCI hotplug add: thunderbolt-attached endpoint after boot resolves to correct DRHD via `dmar_pci_bus_notifier`.
- [ ] AC-6: IOMMU PMU registered: `perf list | grep iommu` shows per-IOMMU PMU source available; `perf stat -e iommu/iotlb_invalidate/` works.
- [ ] AC-7: Spec-violation defense: malformed DMAR table (truncated header, bad checksum) detected + driver bails with error log; no panic.
- [ ] AC-8: VT-d disable test: kernel cmdline `intel_iommu=off` skips DMAR parse + falls back to passthrough mode.

## Architecture

`DmarTable` global structure for per-platform DMAR data:

```
struct DmarTable {
  drhd: KVec<DrhdUnit>,
  rmrr: KVec<RmrrUnit>,
  atsr: KVec<AtsrUnit>,
  satc: KVec<SatcUnit>,
  rhsa: BTreeMap<u64, u32>,           // base_addr → numa-node
  pci_notifier_block: NotifierBlock,
}

struct DrhdUnit {
  reg_base_addr: u64,
  reg_size: u32,
  segment: u16,
  flags: u8,                          // INCLUDE_ALL bit
  dev_scope: KVec<DevScope>,
  iommu: Option<KArc<IntelIommu>>,    // populated by alloc_iommu
  numa_node: i32,                     // from RHSA
}

struct DevScope {
  type_: u8,                          // PCI_ENDPOINT / PCI_SUB_HIERARCHY / IOAPIC / HPET / NAMESPACE
  start_bus: u8,
  path: KVec<DmarPciPath>,            // BDF path-list
  pdev: Option<KWeak<PciDev>>,        // resolved at PCI-bus-add
}

struct RmrrUnit {
  base_addr: u64,
  end_addr: u64,
  dev_scope: KVec<DevScope>,
}
```

`DmarTable::init` flow:
1. `acpi_get_table("DMAR", 0, &table)` (via ACPI subsystem).
2. Validate header: signature "DMAR", checksum, length within ACPI-table bounds.
3. Read DMAR header.haw (host-address-width); init global `dmar_haw`.
4. Walk entries from `table + sizeof(header)` to `table + table.length`:
   - Per-entry: read type from header.type; dispatch:
     - DRHD → `DmarTable::parse_one_drhd`
     - RMRR → `DmarTable::parse_one_rmrr`
     - ATSR → `DmarTable::parse_one_atsr`
     - RHSA → `DmarTable::parse_one_rhsa`
     - ANDD → `DmarTable::parse_one_andd`
     - SATC → `DmarTable::parse_one_satc`
5. Per-DRHD: walk dev-scope from header offset; per-scope build DevScope; push DrhdUnit to global list.
6. After full parse: register PCI bus notifier for hotplug-driven dev-scope resolution.

`DmarTable::dev_scope_init` (called after PCI tree enumerated):
1. For each DrhdUnit + RmrrUnit + AtsrUnit:
   - Per-DevScope: walk PCI tree from segment + start_bus + path; resolve to KWeak<PciDev>.
   - If resolution fails (device hot-removed before init): mark scope as orphan.

`DmarTable::pci_bus_notifier(action, dev)`:
- BUS_NOTIFY_ADD_DEVICE: walk all DRHD/RMRR/ATSR scopes; if scope-BDF matches dev: bind KWeak.
- BUS_NOTIFY_DEL_DEVICE: walk all scopes; clear matching KWeak.

`DrhdUnit::alloc_iommu` flow:
1. ioremap `reg_base_addr` for `reg_size` bytes (typically 4KiB; 8KiB if VT-d3).
2. Read VER (offset 0x00); validate ≥ supported.
3. Read CAP (offset 0x08) + ECAP (offset 0x10); cache in IntelIommu struct.
4. Allocate IntelIommu struct; populate fields.
5. Set IntelIommu.numa_node from DrhdUnit.numa_node.
6. Push to global iommu-list.

`IommuPmu::register(iommu)` (perfmon.c):
1. Allocate `pmu` (perf-event PMU descriptor).
2. Set `pmu.name = "iommu_<id>"`.
3. Set `pmu.event_init = IommuPmu::event_init`.
4. Set `pmu.add` / `pmu.del` / `pmu.start` / `pmu.stop` / `pmu.read`.
5. Register via `perf_pmu_register(&pmu, ...)`.
6. On counter-overflow MSI: per-counter `perf_event_overflow(...)` queued.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dmar_walk_no_oob` | OOB | per-table walk bounded by table.length; per-entry header.length validates ≥ minimum + ≤ remaining-bytes. |
| `dev_scope_walk_no_oob` | OOB | per-dev-scope walk bounded by entry.length; per-path bounded by scope.length-header. |
| `rmrr_range_well_formed` | INVARIANT | per-RMRR base_addr ≤ end_addr; range page-aligned; range fits in physical addr space. |
| `pci_notifier_no_uaf` | UAF | per-DevScope.pdev as KWeak; upgrade-on-use; defense against PCI-bus-del before scope-deinit. |

### Layer 2: TLA+

`drivers/iommu/intel/dmar_dev_scope.tla` models per-DevScope two-phase resolution: state ∈ {Pending, Resolved, Orphan}; transitions via dmar_pci_bus_notifier vs DRHD-detach.

Properties:
- `safety_no_resolved_after_orphan` — once Orphan, no transition back to Resolved without explicit re-init.
- `liveness_pending_resolves` — every Pending eventually Resolved or Orphan; no infinite-pending.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DmarTable::parse` post: all DRHD/RMRR/ATSR/RHSA/SATC entries from table parsed; counts match table-walk | `DmarTable::parse` |
| `DrhdUnit::alloc_iommu` post: per-IOMMU registers ioremapped; IntelIommu populated; pushed to global list | `DrhdUnit::alloc_iommu` |
| Per-DrhdUnit at most one `iommu` field set — defense against double-init | `DrhdUnit::alloc_iommu` |
| Per-PCI-bus-notify: scope.pdev = upgraded(KWeak) iff PCI dev exists | `DmarTable::pci_bus_notifier` |
| RMRR identity-map applied at intel-iommu init for all listed devices | `IntelIommu::apply_rmrr_identity_maps` |

### Layer 4: Verus/Creusot functional

`DmarTable::parse(acpi_table_bytes) → DmarTable struct` parser-correctness: per-entry parsed iff entry header.type ∈ supported-set; entry counts + per-entry fields match byte-level encoding.

## Hardening

(Inherits row-1 features from `drivers/iommu/intel-iommu.md` § Hardening.)

dmar-specific reinforcement:

- **ACPI table checksum validation** + length validation — defense against malformed BIOS table causing parser walk OOB.
- **Per-entry length validation** — defense against truncated entry causing inner-walk OOB.
- **RMRR range validation** — defense against malformed RMRR with end_addr < base_addr or unaligned addr.
- **Per-DRHD reg-base address validation** — must be in ACPI-reserved MMIO range; defense against BIOS pointing to RAM.
- **VT-d version compat check** — defense against unsupported VT-d revision.
- **Per-DevScope BDF validation** — segment/bus/dev/func within valid PCI ranges; defense against BIOS-supplied bogus path.
- **Per-PMU event-validation** — defense against userspace requesting unsupported event raising AB through perf_event_open.
- **Per-IOMMU PMU MSI affinity-locked** — defense against userspace re-affining counter MSI to different CPU breaking lock-acquisition.
- **Per-DMAR-table parse-once** flag — defense against multiple-entry-into init causing double-alloc.
- **Per-PCI-bus-notifier reentrancy guard** — held during scope-update; defense against concurrent PCI-add/del with init.
- **kernel cmdline `intel_iommu=off`** honored — early bail before any DMAR access; defense against forced VT-d on broken hardware.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — DMAR table parse paths consume ACPI-firmware bytes only via bounded helpers; whitelisted slab caches for `dmar_drhd_unit`, `dmar_rmrr_unit`, `dmar_atsr_unit`, `dmar_satc_unit`.
- **PAX_KERNEXEC** — DMAR enumeration and PCI-bus-notifier registration in W^X kernel text; ACPI-table walkers not patchable at runtime.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across DMAR table parse, scope walk, and hotplug PCI-add/del notifier entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `dmar_drhd_unit` and scope-device references; overflow trap defeats hotplug-vs-init races on scope updates.
- **PAX_MEMORY_SANITIZE** — zero-on-free for DMAR unit slabs and scope-device arrays so prior ACPI-derived topology cannot bleed across boot-stage reuse.
- **PAX_UDEREF** — SMAP/PAN enforced; DMAR has no userland data path but any debugfs/sysfs probe attribute coerces canonical helpers.
- **PAX_RAP / kCFI** — DMAR walker callbacks (`dmar_walk_remapping_entries`, scope-handlers, hotplug-notifier) `__ro_after_init` and kCFI-typed.
- **GRKERNSEC_HIDESYM** — gate kallsyms and DMAR-unit pointer disclosure behind CAP_SYSLOG; suppress `%p` in DMAR debug.
- **GRKERNSEC_DMESG** — restrict DMAR parse warnings, RMRR/ATSR-conflict banners, and hotplug scope-update logs to CAP_SYSLOG to deny attackers a free IOMMU topology map.
- **CAP_SYS_ADMIN strict** — debugfs surfaces exposing DMAR enumeration require CAP_SYS_ADMIN; `intel_iommu=` and `dmar=` cmdline parsing locked down behind boot context.
- **RMRR loopback protection** — RMRR ranges policed against overlap with kernel `.text`/`.rodata` and the direct-map; reject ACPI tables that name an RMRR over kernel image.
- **ATS/PASID gating** — ATSR-described ATS-capable endpoints policed against allowlist; ATS not silently enabled for non-allowlisted devices.
- **Hotplug notifier reentrancy** — `pci_bus_notifier` guard held during scope-update; lockdep-asserted against concurrent PCI-add/del during DMAR init.
- **`intel_iommu=off` honoured** — early bail before any DMAR mmio access; defense against forcing VT-d on hardware with known-broken DMAR tables.
- **DMAR parse-once flag** — table parse idempotent; multiple-entry-into-init refused to defeat double-alloc.

Rationale: DMAR is firmware-supplied data that drives the very fabric of IOMMU isolation — a malicious or buggy ACPI table can declare RMRR loopback over kernel `.text`, force-enable ATS on attacker-controlled endpoints, or define overlapping scopes that break domain isolation. Without strict RMRR overlap policing, ATS allowlisting, refcount-overflow trapping on `drhd` units, and CAP_SYS_ADMIN gating on debugfs, the platform's IOMMU topology becomes attacker-influenceable from the supply chain inwards.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Generic IOMMU API (covered in `iommu-core.md` Tier-3)
- Intel IOMMU driver per-domain ops (covered in `intel-iommu.md` Tier-3)
- Intel PASID-table mgmt (covered in `intel-pasid.md` Tier-3)
- IOVA allocator (covered in `iova.md` Tier-3)
- AMD IOMMU + ARM SMMU drivers
- Implementation code
