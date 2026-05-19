# Tier-3: drivers/nvdimm/{bus,dimm,region,namespace_*}.c — DPA→SPA mapping, label storage, namespace activation

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/nvdimm/00-overview.md
upstream-paths:
  - drivers/nvdimm/bus.c
  - drivers/nvdimm/dimm.c
  - drivers/nvdimm/dimm_devs.c
  - drivers/nvdimm/region.c
  - drivers/nvdimm/region_devs.c
  - drivers/nvdimm/namespace_devs.c
  - drivers/nvdimm/label.c
  - drivers/nvdimm/label.h
  - drivers/nvdimm/nd.h
  - drivers/nvdimm/nd-core.h
  - drivers/nvdimm/btt.c
  - drivers/nvdimm/security.c
-->

## Summary

The libnvdimm core (`bus.c` + `dimm.c` + `dimm_devs.c` + `region.c` + `region_devs.c` + `namespace_devs.c` + `label.c` + BTT companion) implements the device-model machinery underneath ndctl(1)/daxctl(1). Its primary job is to maintain a consistent, persistent **Device Physical Address (DPA) → System Physical Address (SPA)** mapping across reboots, capacity reshuffling, and provider unbind/rebind. The mapping lives in two places: (a) the per-DIMM **label storage area** (label-area = a small persistent partition on each DIMM holding `nd_namespace_label` records — EFI v1.1/v1.2 layout for NFIT-class DIMMs, CXL-specific layout for CXL-attached PMEM), and (b) the per-bus **interleave-set descriptor** (a tuple of `(type_guid, cookie1, cookie2, altcookie)` derived from the platform's interleave geometry that verifies DIMM-set integrity at region activation time). Userspace mutates the label area via ND_CMD opcodes; the kernel parses, reserves DPA extents (`ndd->dpa` resource tree), and instantiates `nd_namespace_pmem` / `nd_namespace_io` children whose block-device aliases (`/dev/pmemN`) sit atop the SPA range via the `pmem` driver, or whose BTT/PFN/DAX seed children re-layer the namespace.

This Tier-3 covers the DPA-tree + label-area machinery + namespace-device lifecycle + BTT write-atomicity protocol. Cross-ref `nvdimm/00-overview.md` for subsystem framing and `cxl/00-overview.md` for the CXL-bridge provider.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nvdimm_drvdata` (`ndd`) | per-DIMM driver state (label buffer, DPA tree, ns slot indices, kref) | `drivers::nvdimm::DimmDrvdata` |
| `ndd->data` | in-DRAM cache of label-area contents (size = `nsarea.config_size`) | `DimmDrvdata::data` |
| `ndd->dpa` | per-DIMM DPA resource tree (root = `[0, -1]`, sub-resources per label extent) | `DimmDrvdata::dpa` |
| `ndd->ns_current` / `ndd->ns_next` | active + next slot indices in label rotation | `DimmDrvdata::ns_*` |
| `struct nd_namespace_label` (union: `efi` v1.1/v1.2, `cxl`) | on-DIMM label record | `drivers::nvdimm::Label` |
| `struct nd_namespace_index` | label-area index block (per-slot bitmap, sequence, checksum) | `drivers::nvdimm::LabelIndex` |
| `nd_label_data_init(ndd)` | read label area + validate + cache | `DimmDrvdata::label_data_init` |
| `nd_label_reserve_dpa(ndd)` | build DPA resource tree from labels | `DimmDrvdata::reserve_dpa` |
| `nd_label_active(ndd)` / `nd_label_next_nsindex(ndd)` | resolve current + next index slot | `DimmDrvdata::label_active` |
| `nd_label_validate(ndd)` | validate index checksum + sequence number | `DimmDrvdata::label_validate` |
| `nsl_get_dpa()` / `nsl_get_rawsize()` / `nsl_get_flags()` etc. | label-field accessors w/ EFI-vs-CXL union | `Label::get_*` |
| `nsl_set_dpa()` / `nsl_set_rawsize()` etc. | label-field setters | `Label::set_*` |
| `nd_pmem_namespace_label_update(...)` | flush in-DRAM label changes to DIMM | `Namespace::label_update` |
| `nd_namespace_pmem_set_resource(...)` / `_io_set_resource(...)` | compute SPA range from DPA extent + interleave geometry | `Namespace::set_resource` |
| `arena_info` + `btt_sb` | BTT arena state + on-media superblock | `drivers::nvdimm::Arena` |
| `btt_info_write(arena, super)` / `btt_info_read(...)` | superblock write/read with mirror | `Arena::info_*` |
| `nvdimm_security_unlock(dev)` / `nvdimm_security_change_key(...)` / `_freeze(...)` / `_passphrase_secure_erase(...)` | security state machine | `Dimm::security_*` |
| `nd_cmd_dimm_desc(cmd)` / `nd_cmd_bus_desc(cmd)` | per-cmd in/out element table | `Cmd::desc` |

## Compatibility contract

REQ-1: Label storage area layout (EFI v1.2 + CXL v1.0): two index blocks alternating (`current` + `next`) at the start of the area, each containing a sequence number + checksum + per-slot free bitmap; followed by an array of `nd_namespace_label` slots.

REQ-2: Label record fields (union via `nsl_get_*` helpers): UUID, name (64 B), DPA, rawsize, flags, slot, position-in-set, nrange (multi-range labels for v1.2), interleave-set cookie, checksum.

REQ-3: Per-DIMM DPA resource tree rooted at `[0, U64_MAX]`; per-namespace label extent claimed via `request_resource` as a sub-resource; collision returns `-EBUSY`.

REQ-4: Label update protocol (write-ahead): allocate a free slot in `next` index → write label data into that slot → checksum + sequence-number-bump the `next` index → write `next` index to DIMM → swap `current ⇄ next`. Atomic against power-loss: at any point, exactly one valid index is readable.

REQ-5: Namespace creation: userspace writes to `regionN/create_namespace` → kernel allocates a UUID + reserves DPA extent via `request_resource` on each participating DIMM → writes labels → instantiates `nd_namespace_pmem` child device.

REQ-6: SPA computation: for an interleave-set with stride `S` and `N` participating DIMMs at positions `[0..N)`, the SPA-from-DPA function is `spa(d, pos) = region_spa_base + ((d / S) * N + pos) * S + (d mod S)`; the inverse `dpa(s) = (s - base) - ((s - base) / (N*S)) * (N-1) * S` selects the per-DIMM DPA when `pos` matches.

REQ-7: BTT arena layout (per arena, per power-of-2 `external_lba_size`): superblock at `info_off` + mirror at `info2_off` (both 512-aligned); data area (size = external_nlba × external_lba_size); map area (4 B per external LBA); log area (16 B per concurrent lane); padding to align next arena.

REQ-8: BTT write-atomicity: each external LBA write claims a lane → reads current map entry → writes new data to a free internal LBA → updates log + map atomically (via log entry CRC + sequence) → frees old internal LBA. Power-loss between any two steps leaves the previous LBA contents intact.

REQ-9: Security state machine: per-DIMM flags `DISABLED → UNLOCKED ⇄ LOCKED → FROZEN`; `nvdimm_security_unlock` issues the provider-specific unlock opcode and updates `NDD_LOCKED` on success; `freeze` is one-way and persists across reboot until next sanitize.

REQ-10: ND_CMD ioctl envelope: per-command `nd_cmd_desc` declares `in_num` + `out_num` + per-element sizes; envelope total bounded by `ND_IOCTL_MAX_BUFLEN` (4 MB); variable-length elements bounded by an in-band length field.

REQ-11: Vendor opcode (`ND_CMD_VENDOR`) — opaque vendor payload; per-provider `dimm_family_mask` / `bus_family_mask` declares accepted vendor families; refused opcodes return `-ENOTTY`.

REQ-12: Per-region `flush` callback (e.g. virtio_pmem) invoked at fsync; returns when the device acknowledges durability.

## Acceptance Criteria

- [ ] AC-1: After `ndctl create-namespace -r regionX -s 1G`, the label area on each participating DIMM contains a record with matching UUID, identical interleave-set cookie, and per-DIMM DPA extent that sums to the namespace size.
- [ ] AC-2: Power-cycle between label-slot-write and index-block-write leaves the namespace either fully present or fully absent — never half-present.
- [ ] AC-3: `ndctl destroy-namespace -f` releases the DPA resource tree entries; subsequent `create-namespace` of the same size succeeds at the freed offset.
- [ ] AC-4: BTT torn-write test (qemu pmem-backend with `--persistence=writeback` + power-cut between data write and map write): each LBA reads back as either the pre-write or post-write value, never partial.
- [ ] AC-5: `ndctl setup-passphrase` advances `NDD_LOCKED=0 → set`; subsequent reboot leaves DIMM in `LOCKED`; `unlock` clears `NDD_LOCKED`.
- [ ] AC-6: `ndctl freeze-security` issues freeze opcode; subsequent `setup-passphrase` returns `-EBUSY`; freeze persists across reboot.
- [ ] AC-7: `ND_CMD_VENDOR` with payload exceeding `ND_IOCTL_MAX_BUFLEN` rejected with `-EINVAL`; vendor opcode outside `family_mask` rejected with `-ENOTTY`.

## Architecture

`DimmDrvdata` (per-DIMM, `drivers::nvdimm::DimmDrvdata`):

```
struct DimmDrvdata {
  dev: &'static Device,
  nslabel_size: usize,            // sizeof(nd_namespace_label) per family
  nsarea: NdCmdGetConfigSize,     // { config_size, max_xfer }
  data: KVec<u8>,                 // in-DRAM label-area cache
  cxl: bool,                      // CXL vs NFIT label layout
  ns_current: i32,                // active index slot
  ns_next: i32,                   // shadow index slot
  dpa: Resource,                  // per-DIMM DPA resource tree
  kref: Refcount,
}
```

DIMM probe sequence (cross-ref `nvdimm/00-overview.md`):
1. `nvdimm_security_setup_events(dev)`.
2. `nvdimm_check_config_data(dev)` — issues `GET_CONFIG_SIZE` opcode → fills `ndd->nsarea`.
3. `nvdimm_security_unlock(dev)` if firmware reports `LOCKED`.
4. `nvdimm_init_nsarea(ndd)` — issues chunked `GET_CONFIG_DATA` reads of size `min(max_xfer, config_size - offset)`; assembles `ndd->data`.
5. `nd_label_data_init(ndd)` — validates both index blocks (CRC + sequence) → selects `ns_current` (highest valid sequence); calls `nd_label_reserve_dpa`.
6. `nd_label_reserve_dpa(ndd)` — for each label-area slot with `nsl_get_flags(...) & SLOT_ALLOC`: `request_resource(ndd->dpa, &sub_res)` claiming the `[dpa, dpa+rawsize)` range.

Label update protocol (write-ahead, `nd_label_set_namespace` + `nd_label_write_index`):
1. Acquire `nvdimm_bus` mutex.
2. Allocate free slot in `ns_next` index block from the per-slot bitmap.
3. Build `nd_namespace_label` record in DRAM (per-family via union accessors).
4. Issue chunked `SET_CONFIG_DATA` writes covering the slot bytes.
5. Bump `ns_next` index `seq`, recompute checksum.
6. Issue `SET_CONFIG_DATA` for the index block.
7. Swap `ns_current ⇄ ns_next` in DRAM (next reboot re-reads index, picks higher seq).

Region + namespace activation (`nd_region_activate` → `nd_region_register_namespaces`):
1. For each mapping: cross-correlate labels by UUID; verify all required positions present (`nr_positions == iw`).
2. Verify per-namespace label cookie matches `nd_set->cookie1` (v1.1) or `cookie2` (v1.2).
3. Allocate `nd_namespace_pmem` device:
   - `nspm->uuid`, `nspm->alt_name`.
   - `nspm->nsio.res` = computed SPA range.
4. `nd_namespace_pmem_set_resource` invokes the DPA→SPA function (region_spa_base + interleave math).
5. Register child device; `nd_pmem_driver` binds and exposes `/dev/pmemN`.

SPA computation (`region_devs.c`, simplified):
- For pure (single-DIMM) region: `spa = region_base + dpa`.
- For interleave set of `iw` DIMMs at stride `ig`: `spa = region_base + (dpa / ig) * (iw * ig) + position * ig + (dpa mod ig)` where `position` is the DIMM's position-in-set.
- Bounded by `region->ndr_size`; refused if any per-DIMM DPA extent maps outside the region's SPA window.

BTT bio submission (`drivers/nvdimm/btt.c`):
1. Per-bio split into per-LBA operations.
2. For each LBA:
   - Acquire a lane (lock-free per-CPU lane pool of size `ND_MAX_LANES`).
   - Read map[external_lba] → current internal_lba.
   - For write: allocate free internal_lba from free pool; write data via `arena_write_bytes` (uses `nvdimm_write_bytes` → CLWB/SFENCE or virtio async-flush).
   - Update log entry (old_internal, new_internal, external_lba, seq); flush.
   - Atomically swap map[external_lba] = new_internal; flush.
   - Return old_internal to free pool.
   - Release lane.

Security state machine (`security.c`):
- `set_pass`: opcode `SET_PASSPHRASE` with old+new passphrase; on success, DIMM enters `LOCKED` at next power cycle.
- `unlock`: opcode `UNLOCK` with passphrase; on success, `NDD_LOCKED` cleared.
- `freeze`: opcode `FREEZE_SECURITY`; no further passphrase ops accepted until next sanitize.
- `secure_erase` (passphrase-secure-erase or sanitize): opcode `PASSPHRASE_SECURE_ERASE`; on success, DIMM contents zeroed + passphrase reset; `cpu_cache_invalidate_all` runs before next region activation.

## Hardening

- **Label-area CRC + sequence verified** — `nd_label_validate` rejects index blocks with bad checksum or non-monotonic sequence; defense against truncated writes or storage-side bit-rot.
- **Index-block ping-pong write-ahead** — `current` + `next` alternation guarantees atomicity across power loss; defense against half-written labels.
- **DPA resource tree `request_resource`** — sub-resource claims fail on overlap; defense against namespace-overlap UAF.
- **Cookie verification on activation** — region refuses to bind namespace whose label cookie does not match interleave-set cookie; defense against accidental cross-set namespace import.
- **Chunked label transfer** — bounded by `min(max_xfer, config_size - offset)`; defense against single-transaction-too-big errors.
- **BTT lane pool bounded** — `ND_MAX_LANES = 256`; defense against unbounded lane allocation under heavy parallel I/O.
- **BTT superblock + mirror** — both written on every config change; arena-recovery reads mirror on primary checksum failure.
- **Per-cmd `nd_cmd_desc` envelope** — IOCTL refuses opcodes with mismatched in/out sizes or out-of-table opcode.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `nvdimm_drvdata`, `nd_namespace_pmem`, `nd_namespace_io`, `nd_btt`, `arena_info`, label-buffer allocations; reject userspace copies outside `nd_ioctl` helpers; in particular `GET_CONFIG_DATA` / `SET_CONFIG_DATA` payloads policed against `nsarea.config_size`.
- **PAX_KERNEXEC** — libnvdimm DPA/label code (`nd_label_data_init`, `nd_label_reserve_dpa`, `nd_label_write_index`, `nd_region_register_namespaces`, BTT bio submission, security state transitions) in `__ro_after_init` text with W^X enforced.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `nd_ioctl`, BTT bio submission, label-write, security-state transition entries.
- **PAX_REFCOUNT** — saturating `refcount_t` (kref) on `nvdimm_drvdata` (`put_ndd`), `nd_namespace_*`, `arena_info`, `nd_btt`; overflow trap defeats unbind-vs-IOCTL race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `ndd->data` (label-area cache), passphrase staging in `cxl_set_pass`/`cxl_disable_pass`/`cxl_pass_erase` and the NFIT equivalents, BTT log/map shadow buffers, namespace `alt_name`/UUID strings so stale labels and plaintext passphrases never bleed into a reused slab object.
- **PAX_UDEREF** — SMAP/PAN enforced on every `nd_ioctl` and per-DIMM cdev entry; refuse user-pointer deref outside the canonical envelope helpers.
- **PAX_RAP / kCFI** — `nd_cmd_desc` dispatch, `nvdimm_security_ops` (`get_flags`, `change_key`, `disable`, `freeze`, `unlock`, `erase`, `disable_master`), BTT bio-end-io, region `flush` callback marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-DIMM / per-namespace / per-arena pointer disclosure behind CAP_SYSLOG; suppress `%p` in libnvdimm tracepoints.
- **GRKERNSEC_DMESG** — restrict cookie-mismatch, index-block-CRC-fail, BTT-superblock-parity-fail, security-state-transition, passphrase-buffer-leak banners to CAP_SYSLOG.
- **ND_CMD requires CAP_SYS_ADMIN** — every ND_CMD opcode that mutates persistent state (`SET_CONFIG_DATA`, `ARS_START`, `CLEAR_ERROR`, security ops, vendor) gated by CAP_SYS_ADMIN in the owning user namespace; even read-only opcodes (`GET_CONFIG_DATA`, `SMART`) refused from non-init userns by default.
- **Vendor-cmd allowlist** — `ND_CMD_VENDOR` opcodes refused unless explicitly in `dimm_family_mask` / `bus_family_mask`; payload size bounded by per-family table.
- **Security state machine enforced in kernel** — userspace cannot transition FROZEN → DISABLED without sanitize completion notification; `secure_erase` postcondition includes `cpu_cache_invalidate_all` before region reactivation.
- **BTT write-atomicity** — every log+map+data triple sequenced with explicit flush barriers; refuse to commit an LBA without log entry CRC valid; refuse partial-arena activation on superblock mismatch.
- **DPA tree audit** — every `request_resource` failure logged with `dev_warn_ratelimited`; every successful claim emits a structured audit record (DIMM, DPA, rawsize, UUID) for CAP_AUDIT_WRITE consumers.

Rationale: libnvdimm's label area is the persistent ground truth for who owns which byte of DIMM; a relaxed cookie check, a non-zeroed passphrase staging buffer, a missed CRC verification, or a partial label update creates an exploitable cross-namespace aliasing or passphrase-disclosure primitive. CAP_SYS_ADMIN on every mutating ND_CMD, PAX_USERCOPY on label/passphrase paths, MEMORY_SANITIZE on plaintext passphrase + label buffers, vendor-cmd family allowlist, BTT log+map sequenced barriers, and security state-machine refinement make persistent-memory operation a mediated boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `label_xfer_bounded` | OOB | every `GET_CONFIG_DATA` / `SET_CONFIG_DATA` chunk satisfies `offset + length ≤ ndd->nsarea.config_size`. |
| `dpa_tree_no_overlap` | UNIQUENESS | post `nd_label_reserve_dpa`: no two sub-resources in `ndd->dpa` overlap. |
| `btt_lane_no_oob` | OOB | per-CPU lane index < `ND_MAX_LANES`; lane pool free-list head bounded. |
| `passphrase_no_leak` | INFO-LEAK | passphrase staging buffer zeroed before slab return on every path (success + error). |

### Layer 2: TLA+

`models/nvdimm/label_ping_pong.tla` (parent-declared): models alternating `current` + `next` index writes under arbitrary power-loss; proves at every state exactly one valid index is readable + that consumers always select the higher-sequence valid index.

`models/nvdimm/btt_lba.tla` (parent-declared): models BTT log+map+data triple per external LBA under power loss; proves each external LBA always reads either pre-write or post-write data, never partial.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `nd_label_data_init` post: `ndd->ns_current ∈ {0, 1}` AND `nd_label_active(ndd)` returns the higher-sequence valid index. | `nvdimm/label` |
| `nd_namespace_pmem_set_resource` post: `nspm->nsio.res` ⊆ `nd_region->ndr_start + ndr_size`. | `nvdimm/namespace_devs` |
| `nvdimm_security_change_key` post: on `Ok`, old + new staging buffers zeroed; on `Err`, same. | `nvdimm/security` |
| BTT bio post: `arena.map[external_lba]` reflects committed write OR remained unchanged on failure. | `nvdimm/btt` |

### Layer 4: Verus/Creusot functional

End-to-end: `nd_ioctl(ND_IOCTL, ND_CMD_SET_CONFIG_DATA, label_buffer)` → chunked `SET_CONFIG_DATA` over mailbox → on-DIMM label-area updated atomically via index-block ping-pong → next region activation reconstructs the same UUID → DPA → SPA → block-device tuple; encoded as a Verus refinement from IOCTL envelope to persistent-state transition.

## Out of Scope

- ACPI NFIT table parsing (covered in `drivers/acpi/nfit/00-overview.md` future Tier-3)
- CXL nvdimm-bridge integration details (covered in `cxl/00-overview.md`)
- PFN device mode + struct page layout (covered in `mm/memremap.md` future Tier-3)
- virtio_pmem async-flush protocol (covered in `drivers/virtio/virtio_pmem.md` future Tier-3)
- per-vendor SMART/ARS opcode minutiae
