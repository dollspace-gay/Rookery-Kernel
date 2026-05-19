# Tier-3: drivers/nvdimm/ + drivers/acpi/nfit/ — libnvdimm subsystem (namespaces, BTT, sector mode)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/nvdimm/00-overview.md
upstream-paths:
  - drivers/nvdimm/bus.c
  - drivers/nvdimm/core.c
  - drivers/nvdimm/dimm.c
  - drivers/nvdimm/dimm_devs.c
  - drivers/nvdimm/region.c
  - drivers/nvdimm/region_devs.c
  - drivers/nvdimm/namespace_devs.c
  - drivers/nvdimm/label.c
  - drivers/nvdimm/btt.c
  - drivers/nvdimm/btt_devs.c
  - drivers/nvdimm/claim.c
  - drivers/nvdimm/pmem.c
  - drivers/nvdimm/security.c
  - drivers/nvdimm/badrange.c
  - drivers/acpi/nfit/core.c
  - include/linux/libnvdimm.h
  - include/uapi/linux/ndctl.h
-->

## Summary

libnvdimm is the Linux subsystem for non-volatile DIMM and CXL-attached persistent memory: it abstracts platform-specific persistent-memory enumeration (ACPI NFIT for legacy NVDIMM-N / DDR-T, CXL CEDT/CFMWS bridge for CXL PMEM, OF device-tree for embedded `of_pmem`, virtio_pmem for guest-paravirtualized PMEM) into a uniform tree of `nvdimm_bus` → `nvdimm` (DIMM) + `nd_region` (interleave set) → `nd_namespace_*` (PMEM / BLK / IO) → optional `nd_btt` (Block Translation Table for sector-mode write-atomicity) / `nd_pfn` (struct-page over PMEM) / `nd_dax` (devdax over PMEM). Each layer is a struct-device on the `nvdimm_bus_type`, bound by per-type drivers (`nvdimm_driver`, `nd_region_driver`, `nd_pmem_driver`, `nd_btt_driver`, etc.).

Namespaces are addressed in **Device Physical Address (DPA)** space on each DIMM and mapped into **System Physical Address (SPA)** space by the interleave set's stride/granularity (NFIT SPA range mapping or CXL HDM decoder topology). On-DIMM **namespace labels** (EFI v1.2 layout for NFIT-class, CXL label layout for CXL-class) carry the persistent identifier, DPA extent, interleave-set cookie, and namespace type so namespaces survive reboot / module reload / capacity reshuffling. **BTT** layers an atomic-sector translation on top of an arbitrary PMEM namespace so that the sector-granularity write-atomicity required by classic filesystems (ext4 without DAX, XFS without DAX, swap) is preserved on byte-addressable media that has no inherent sector primitive.

This Tier-3 covers the subsystem framing: bus / class / per-stratum drivers, namespace + label semantics, BTT + pfn + dax sub-types, security state machine, and the ND_CMD / vendor-cmd UAPI surface. Library-level DPA→SPA mapping + label parsing + namespace activation details are in `nvdimm/libnvdimm.md`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nvdimm_bus` | bus container (per-provider) | `drivers::nvdimm::Bus` |
| `struct nvdimm_bus_descriptor` | per-provider ops table (NFIT, CXL, OF, virtio) | `drivers::nvdimm::BusDescriptor` |
| `struct nvdimm` | per-DIMM device | `drivers::nvdimm::Dimm` |
| `struct nvdimm_drvdata` (`ndd`) | per-DIMM driver state (labels, DPA tree, security) | `drivers::nvdimm::DimmDrvdata` |
| `struct nd_region` | interleave-set device | `drivers::nvdimm::Region` |
| `struct nd_region_data` (`ndrd`) | per-region driver state | `drivers::nvdimm::RegionData` |
| `struct nd_mapping` / `nd_mapping_desc` | DIMM-position within interleave set | `drivers::nvdimm::Mapping` |
| `struct nd_namespace_pmem` / `_io` | PMEM / IO namespace | `drivers::nvdimm::Namespace*` |
| `struct nd_namespace_label` (EFI / CXL union) | on-DIMM label record | `drivers::nvdimm::Label` |
| `struct nd_btt` / `arena_info` / `btt_sb` | BTT instance / arena / superblock | `drivers::nvdimm::Btt*` |
| `struct nd_pfn` / `nd_dax` | pfn-mode / dax-mode child devices | `drivers::nvdimm::Pfn`, `::Dax` |
| `struct badrange` / `badrange_entry` | per-bus poison range tracking | `drivers::nvdimm::Badrange` |
| `nvdimm_bus_register(parent, desc)` / `_unregister(bus)` | bus lifecycle | `Bus::register` / `_unregister` |
| `nvdimm_create(bus, ...)` / `__nvdimm_create(...)` | DIMM register | `Dimm::create` |
| `nvdimm_pmem_region_create(bus, desc)` | PMEM region register | `Region::create_pmem` |
| `nd_region_activate(nd_region)` | per-region activation (interleave-set cookie check) | `Region::activate` |
| `nd_label_data_init(ndd)` / `nd_label_reserve_dpa(ndd)` | read + parse + reserve labels | `Dimm::label_init` / `_reserve_dpa` |
| `nvdimm_security_setup_events(dev)` / `_unlock(dev)` / `_freeze(...)` | security state machine | `Dimm::security_*` |
| `nd_ioctl(file, cmd, arg)` / `nvdimm_ioctl(...)` | ND_CMD UAPI dispatch | `Bus::ioctl` / `Dimm::ioctl` |
| `nvdimm_provider_data(nvdimm)` | per-DIMM provider-private pointer (e.g. `cxl_nvdimm`) | `Dimm::provider_data` |

## Compatibility contract

REQ-1: `/dev/ndctlN` cdev per bus and `/dev/nmemX` cdev per DIMM expose `ND_IOCTL` (`uapi/linux/ndctl.h`) commands; per-bus + per-DIMM cmd masks describe which commands the provider implements.

REQ-2: Per-bus commands include `ARS_CAP`, `ARS_START`, `ARS_STATUS`, `CLEAR_ERROR`, `CALL` (vendor) per `enum nd_cmd_ars_*`; per-DIMM commands include `SMART`, `SMART_THRESHOLD`, `DIMM_FLAGS`, `GET_CONFIG_SIZE`, `GET_CONFIG_DATA`, `SET_CONFIG_DATA`, `VENDOR`, `CALL`.

REQ-3: Namespace label area accessed via paired `GET_CONFIG_SIZE` + `GET_CONFIG_DATA` + `SET_CONFIG_DATA` opcodes; transfer chunked by `ndd->nsarea.max_xfer`; total area bounded by `ndd->nsarea.config_size`.

REQ-4: Vendor opcodes pass per-bus `vendor_cmd` opcode + opaque payload; per-bus `dimm_family_mask` / `bus_family_mask` declares which vendor families are honored.

REQ-5: Region activation requires interleave-set cookie match against on-DIMM labels: `nd_set->cookie1` (v1.1) or `cookie2` (v1.2) must equal the recorded `interleave_set_cookie` across all participating DIMMs.

REQ-6: PMEM namespace activation requires `NSLABEL_FLAG_LOCAL` set (PMEM mode) and (for v1.2 EFI labels) per-namespace UUID; multiple label slots may chain to form a non-contiguous DPA extent.

REQ-7: BTT instance state: per-arena 4 KiB superblock at `infooff` (and mirror at `info2off`) — magic, parent UUID, external/internal LBA size, arena count, map+log offsets, padding-bounded; superblock parity-checked on each write.

REQ-8: Security state machine per DIMM advances `DISABLED → UNLOCKED ⇄ LOCKED → FROZEN`; passphrase ops (`set_pass`, `unlock`, `freeze`, `secure_erase`) gated by `NDD_LOCKED` + `NDD_SECURITY_OVERWRITE` flags.

REQ-9: Badrange tracking via per-bus `struct badrange`: SPA ranges harvested from ARS + GHES + uncorrectable MCEs; surfaced as `badblocks` per `nd_region` and per namespace block device.

REQ-10: NDD flags: `NDD_UNARMED` (writes may not persist), `NDD_LOCKED` (capacity inaccessible), `NDD_SECURITY_OVERWRITE` (in-progress wipe), `NDD_LABELING` (label-aware DIMM), `NDD_INCOHERENT` (requires cpu_cache_invalidate before next region activation).

REQ-11: PFN / DAX child devices layer `struct page` (in-DRAM or in-PMEM via `--map=mem`/`--map=dev`) over a PMEM namespace so that fsdax + devdax + kmem-onlining can address PMEM as DRAM-equivalent backing.

REQ-12: virtio_pmem provider issues async-flush requests via virtio-ring instead of CLWB/SFENCE; `ND_REGION_ASYNC` flag exposes the asynchronous flush semantic.

## Acceptance Criteria

- [ ] AC-1: `ndctl list -v` enumerates bus, DIMMs, regions, and namespaces with consistent DPA + SPA + interleave-set-cookie fields.
- [ ] AC-2: `ndctl create-namespace -r regionX -m raw` allocates a PMEM namespace whose label survives reboot.
- [ ] AC-3: `ndctl create-namespace -m sector` activates a BTT instance; `dd` of a power-of-2 LBA size completes without torn-sector visible after async powercycle (qemu pmem-backend test).
- [ ] AC-4: `ndctl create-namespace -m fsdax` produces `/dev/pmemN` with `IS_DAX(inode)` honored by ext4/XFS DAX mount.
- [ ] AC-5: `ndctl create-namespace -m devdax` produces `/dev/daxN.M` consumable by `daxctl reconfigure-device --mode=system-ram` (cross-ref `dax/dax-core.md`).
- [ ] AC-6: Security: `ndctl setup-passphrase`, `ndctl freeze-security`, `ndctl sanitize-dimm` advance the state machine in order; passphrase persists across reboot; freeze refuses subsequent set-passphrase.
- [ ] AC-7: Injected ARS error (qemu pmem error-injection) populates `/sys/bus/nd/devices/regionN/badblocks` and is consumed by `pmem` block-device errlist.

## Architecture

The subsystem tree:

```
nvdimm_bus_type
  +-- nvdimm_bus0 (NFIT or CXL or virtio or of_pmem)
  |     +-- nmem0 (struct nvdimm — DIMM 0)
  |     +-- nmem1
  |     +-- region0 (interleave-set, mode=pmem)
  |     |     +-- mapping0 (nmem0 @ DPA[0, 8GB))
  |     |     +-- mapping1 (nmem1 @ DPA[0, 8GB))
  |     |     +-- namespaceN.0 (uuid, mode={raw|sector|fsdax|devdax})
  |     |           +-- btt0 (if sector mode)         → /dev/pmemN
  |     |           +-- pfn0 (if fsdax mode)          → /dev/pmemN with struct page
  |     |           +-- dax0 (if devdax mode)         → /dev/daxN.M
  |     +-- region1 ...
```

Per-provider binders:
- **acpi_nfit_init**: parses ACPI NFIT (System Physical Address Range, Memory Device to SPA Mapping, NVDIMM Control Region, Interleave, Flush-Hint, Platform-Capabilities) → registers `nvdimm_bus` + DIMMs + regions via `nfit_bus_desc`.
- **cxl_pmem (drivers/cxl/pmem.c)**: registers a CXL nvdimm bridge per memdev with persistent partition; uses `cxl_security_ops` + CXL mailbox for DIMM ops.
- **of_pmem**: device-tree node `compatible = "pmem-region"` triggers DT-based bus + region create.
- **virtio_pmem**: virtio device probe registers a single virtio-backed region with async-flush capability.

Bus dispatch (`nvdimm_bus_probe` in `bus.c`):
1. `to_nd_device_driver(dev->driver)` selects per-stratum driver (`nvdimm_driver` for DIMM, `nd_region_driver` for region, `nd_pmem_driver` for PMEM namespace, `nd_btt_driver` for BTT, etc.).
2. `to_bus_provider(dev)` pins provider module.
3. `nvdimm_bus_probe_start` bumps `probe_active`; provider probe invoked.
4. On success of region probe: `nd_region_advance_seeds` activates "seed devices" — pre-allocated child slots that userspace observes as latent BTT/PFN/DAX seeds available to be claimed by namespace creation.

DIMM probe (`drivers/nvdimm/dimm.c`):
1. `nvdimm_security_setup_events` registers a sysfs event-source for security state transitions.
2. `nvdimm_check_config_data` queries `GET_CONFIG_SIZE`.
3. `nvdimm_security_unlock` attempts unlock if firmware reports `LOCKED`.
4. `nvdimm_init_nsarea(ndd)` reads namespace label area into `ndd->data` (size `nsarea.config_size`).
5. `nd_label_data_init(ndd)` parses labels into per-namespace records; on EACCES → `NDD_LOCKED`.
6. `nd_label_reserve_dpa(ndd)` builds `ndd->dpa` resource tree from label DPA extents.

Region probe (`drivers/nvdimm/region.c`):
1. `nd_region_activate(nd_region)` — verifies interleave-set cookie across mappings; allocates per-region flush hints.
2. `nvdimm_badblocks_populate` seeds `nd_region->bb` from per-bus badrange list.
3. `nd_region_register_namespaces` iterates label-derived namespaces, registers each as child.
4. Seed devices created: `nd_btt_create`, `nd_pfn_create`, `nd_dax_create` — placeholder children waiting for user reconfiguration.

Region teardown (`nd_region_remove`):
1. Recursively unregister all child devices.
2. Null out `ns_seed`, `btt_seed`, `pfn_seed`, `dax_seed`.
3. If `cpu_cache_has_invalidate_memregion()`: `cpu_cache_invalidate_all()` to prevent stale-cacheline reuse on next region create over the same SPA.

UAPI surface (`uapi/linux/ndctl.h`):
- `ND_IOCTL_*` opcodes: `ARS_CAP`, `ARS_START`, `ARS_STATUS`, `CLEAR_ERROR`, `SMART`, `SMART_THRESHOLD`, `DIMM_FLAGS`, `GET_CONFIG_SIZE`, `GET_CONFIG_DATA`, `SET_CONFIG_DATA`, `VENDOR`, `CALL`.
- Bounds: `ND_IOCTL_MAX_BUFLEN = 4 MB`; `ND_CMD_MAX_ELEM = 5`; per-command in/out envelope validated against `nd_cmd_desc`.

## Hardening

- **Label-area transfer chunked** — `GET_CONFIG_DATA` / `SET_CONFIG_DATA` use `ndd->nsarea.max_xfer`-sized chunks; total bounded by `ndd->nsarea.config_size`.
- **Interleave-set cookie verification** — region activation refuses if any DIMM's label cookie mismatches; defense against partial-rebuild data corruption.
- **NDD_LOCKED gate** — locked DIMMs allow label enumeration but block region activation; defense against unlocking via region create.
- **NDD_INCOHERENT cache invalidate** — region activation invokes `cpu_cache_invalidate_all` when incoherent flag set; defense against stale-cacheline persisting across reuse.
- **Probe lifecycle interlock** — `probe_active` counter ensures pending probes complete before unbind; defense against probe-vs-remove UAF.
- **Per-bus + per-DIMM cmd mask** — `bus_desc->cmd_mask` and per-DIMM cmd_mask bound which opcodes pass through; defense against unsupported vendor opcode invocation.
- **ND_IOCTL_MAX_BUFLEN cap** — 4 MB ceiling on user buffers; defense against IOCTL allocation DoS.
- **Badrange harvest from ARS + GHES** — error sources aggregated centrally; defense against per-source error drift.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `nvdimm_bus`, `nvdimm`, `nvdimm_drvdata` (label buffer), `nd_region`, `nd_namespace_*`, `nd_btt`, `arena_info`, `nd_pfn`, `nd_dax`; reject userspace copies outside `nd_ioctl` / cdev helpers.
- **PAX_KERNEXEC** — libnvdimm bus core (`nvdimm_bus_probe`, `nd_ioctl`, `nd_region_activate`, `nd_label_data_init`, `nvdimm_security_unlock`) in `__ro_after_init` text with W^X enforced.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `nd_ioctl`, security-state-transition handlers, ARS callbacks, BTT bio submission.
- **PAX_REFCOUNT** — saturating `refcount_t` (kref) on `nvdimm_drvdata` (`put_ndd`), `nd_region`, `nd_namespace_*`, `nd_btt`; overflow trap defeats probe/remove races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for label buffers (`ndd->data`), security-passphrase staging, BTT log/map shadow, namespace name/alt-name strings so stale persistent identifiers, plaintext passphrases, and label contents cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every `nd_ioctl` entry and per-DIMM cdev; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `nvdimm_bus_descriptor.ndctl`, `nvdimm_security_ops`, `nd_region_desc.flush`, BTT bio dispatch, virtio_pmem flush callback marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-DIMM / per-region pointer disclosure behind CAP_SYSLOG; suppress `%p` in nvdimm tracepoints.
- **GRKERNSEC_DMESG** — restrict cookie-mismatch, label-area-EACCES, security-state-transition, ARS-poison-found, BTT-superblock-parity-fail banners to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict on label store** — every `SET_CONFIG_DATA` IOCTL requires CAP_SYS_ADMIN in the owning user namespace; refuse from non-init userns.
- **PAX_MEMORY_SANITIZE on security passphrase** — passphrase staging buffers and on-DIMM-locked extents zero-on-free; `passphrase_secure_erase` must drive `cpu_cache_invalidate_all` before subsequent reuse.
- **Sector-mode BTT integrity** — superblock parity-checked on every write; mirror at `info2off` cross-verified; refuse to activate on parity mismatch.
- **Vendor-cmd allowlist** — `vendor_cmd` opcodes refused unless their `dimm_family` / `bus_family` is in the per-provider `family_mask`; defense against undocumented opcode classes used as covert channels.
- **`SECURITY_OVERWRITE` gate** — DIMMs with overwrite in progress refuse region attach + namespace probe until completion notification fires.

Rationale: libnvdimm controls persistent storage that survives reboot and that mediates passphrase-based at-rest encryption. A relaxed cookie check, a non-zeroed passphrase buffer, an unbounded `SET_CONFIG_DATA`, or a missed cache-invalidate on `NDD_INCOHERENT` lets one workload reuse another's keys or labels. CAP_SYS_ADMIN on label store, PAX_MEMORY_SANITIZE on passphrase + label memory, BTT superblock parity verification, vendor-cmd allowlisting, and sanitize-vs-reuse interlock turn persistent-memory enumeration into a mediated boundary rather than a trust-the-firmware passthrough.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `label_xfer_no_oob` | OOB | `GET_CONFIG_DATA` / `SET_CONFIG_DATA` chunked transfers bounded by `min(max_xfer, config_size - offset)`. |
| `ioctl_envelope_total` | TOTALITY | `nd_ioctl` rejects any opcode whose envelope size exceeds `ND_IOCTL_MAX_BUFLEN` or whose declared in/out sizes mismatch `nd_cmd_desc`. |
| `ndd_kref_no_uaf` | UAF | `put_ndd` releases when kref hits 0; cdev opens hold reference. |
| `cookie_match_total` | TOTALITY | `nd_region_activate` total only when every mapping's label cookie equals `nd_set->cookie{1,2}`. |

### Layer 2: TLA+

`models/nvdimm/security_state.tla` (parent-declared): models `DISABLED → UNLOCKED ⇄ LOCKED → FROZEN` with concurrent `set_pass / unlock / freeze / secure_erase`; proves no transition skips FROZEN (passphrase ops blocked once frozen) and that `secure_erase` followed by region create observes `cpu_cache_invalidate_all`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `nd_label_reserve_dpa` post: `ndd->dpa` resource tree covers every recorded label DPA extent without overlap. | `nvdimm/label` |
| `nd_region_activate` post: every mapping's DIMM is unlocked OR region activation fails with `-EACCES`. | `nvdimm/region` |
| `nvdimm_security_unlock` post: on success `NDD_LOCKED` cleared; on failure flag preserved + passphrase buffer zeroed. | `nvdimm/security` |

### Layer 4: Verus/Creusot functional

End-to-end: `ndctl create-namespace -m sector` → namespace UUID labeled + DPA extent reserved + BTT child registered + `/dev/pmemN` block device exposed with sector-atomic write semantics; encoded as a Verus refinement from the user invocation to on-DIMM label state.

## Out of Scope

- DPA→SPA mapping math + label parsing details (covered in `nvdimm/libnvdimm.md`)
- DAX onlining details (covered in `dax/dax-core.md`)
- ACPI NFIT table parsing internals (covered in `drivers/acpi/nfit/00-overview.md` future Tier-3)
- CXL bridge details (covered in `cxl/00-overview.md`)
- Per-vendor NVDIMM-N controller-specific opcodes (Intel DSM, Microsoft DSM, HPE DSM)
