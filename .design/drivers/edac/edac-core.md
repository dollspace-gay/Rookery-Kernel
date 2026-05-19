# Tier-3: drivers/edac/{edac_mc,edac_device,edac_mc_sysfs,edac_device_sysfs}.c — ECC error reporting framework, MC + L1/L2/L3 device, sysfs surface

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/edac/00-overview.md
upstream-paths:
  - drivers/edac/edac_mc.c
  - drivers/edac/edac_mc.h
  - drivers/edac/edac_mc_sysfs.c
  - drivers/edac/edac_device.c
  - drivers/edac/edac_device.h
  - drivers/edac/edac_device_sysfs.c
  - drivers/edac/edac_module.c
  - drivers/edac/edac_module.h
  - drivers/edac/edac_pci.c
  - drivers/edac/edac_pci_sysfs.c
  - drivers/edac/debugfs.c
  - drivers/edac/wq.c
  - drivers/edac/ghes_edac.c
  - include/linux/edac.h
  - include/ras/ras_event.h
-->

## Summary

The Error Detection And Correction (EDAC) subsystem is the kernel's vendor-agnostic ECC-error reporting framework. It normalizes ECC events from a wide variety of memory-controller, last-level-cache, on-die-cache, and PCI-parity error sources into three uniform device classes: **`mem_ctl_info` (MC)** — a memory-controller with a layered DIMM topology (cs-row / channel / DIMM, or branch / channel / DIMM, or socket / iMC / channel / dimm depending on `n_layers`); **`edac_device_ctl_info` (EDAC device)** — a generic ECC-protected non-RAM block (L1/L2/L3 cache, fabric-level cache, GPU-HBM, FPGA on-die ECC, NIC TCAM) modeled as `instances × blocks` with per-block CE/UE counters; and **`edac_pci_ctl_info` (EDAC-PCI)** — per-bus PCI parity / SERR aggregator. Each class is registered with the EDAC core, exposes a uniform sysfs surface under `/sys/devices/system/edac/`, emits structured `mc_event` + `aer_event` + `non_standard_event` tracepoints (`include/ras/ras_event.h`), and feeds `rasdaemon` / `mcelog` consumers.

The core enforces a small set of policy controls: `edac_mc_panic_on_ue` (panic on any uncorrectable error), `edac_mc_log_ce` / `_log_ue` (gate kernel log output), and `edac_mc_poll_msec` (polling interval for poll-mode drivers); operation modes are `EDAC_OPSTATE_POLL` (driver provides `edac_check()` called from the EDAC workqueue), `_NMI` (NMI / MCE-via-IPI delivery), and `_INT` (per-MC interrupt). Error-injection debugfs hooks (`drivers/edac/debugfs.c`) allow synthetic CE/UE events for testing rasdaemon pipelines without touching hardware.

This Tier-3 covers `edac_mc.c` (MC alloc/register/dispatch + error-desc reporting), `edac_mc_sysfs.c` (per-MC sysfs surface + module params), `edac_device.c` + `edac_device_sysfs.c` (generic ECC device class), `edac_module.c` (subsystem init + workqueue), and the GHES bridge `ghes_edac.c` that lifts firmware-reported ACPI APEI errors into the MC class.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mem_ctl_info` (`mci`) | per-MC control block (layers, csrows, dimms, op-state) | `drivers::edac::MemCtlInfo` |
| `struct dimm_info` | per-DIMM record (label, location, nr_pages, grain, dtype, edac_mode) | `drivers::edac::DimmInfo` |
| `struct csrow_info` / `rank_info` | chip-select-row / channel records | `drivers::edac::CsRow`, `::Rank` |
| `struct edac_raw_error_desc` | normalized error event payload (type, address, syndrome, label, count) | `drivers::edac::RawErrorDesc` |
| `struct edac_device_ctl_info` | generic ECC device control block (instances × blocks) | `drivers::edac::DeviceCtlInfo` |
| `struct edac_device_instance` / `edac_device_block` | per-instance + per-block records | `drivers::edac::DeviceInstance`, `::DeviceBlock` |
| `struct edac_pci_ctl_info` | EDAC-PCI control block | `drivers::edac::PciCtlInfo` |
| `edac_mc_alloc(num, n_layers, layers, sz_pvt)` / `edac_mc_free(mci)` | MC allocate/free | `MemCtlInfo::alloc` / `_free` |
| `edac_mc_add_mc(mci)` / `edac_mc_del_mc(dev)` | MC register/unregister | `MemCtlInfo::add` / `_del` |
| `edac_mc_handle_error(type, mci, count, page, offset, syndrome, ...)` | normalize + dispatch error | `MemCtlInfo::handle_error` |
| `edac_raw_mc_handle_error(e)` | direct path used by ghes_edac + AMD MCE decoder | `MemCtlInfo::raw_handle_error` |
| `edac_device_alloc_ctl_info(pvt_sz, name, instances, blk_name, blocks, off, idx)` | device class alloc | `DeviceCtlInfo::alloc` |
| `edac_device_add_device(dev_ctl)` / `_del_device(dev)` | device class register/unregister | `DeviceCtlInfo::add` / `_del` |
| `edac_device_handle_ce(dev_ctl, instance, block, msg)` / `_handle_ue(...)` | per-block error report | `DeviceCtlInfo::handle_*` |
| `edac_pci_alloc_ctl_info(...)` / `edac_pci_add_device(...)` | EDAC-PCI register | `PciCtlInfo::alloc` / `_add` |
| `edac_mc_get_log_ce()` / `_log_ue()` / `_panic_on_ue()` / `_poll_msec()` | policy getters | `Policy::*` |
| `opstate_init()` | normalize `edac_op_state` to POLL if invalid | `Policy::opstate_init` |
| `edac_workqueue` | shared workqueue for poll-mode drivers | `Subsystem::workqueue` |
| `ghes_edac_register_mc(dev, num, name)` / `_unregister(dev)` / `ghes_edac_report_mem_error(...)` | APEI bridge | `Ghes::register_mc` / `_report` |
| `trace_mc_event(...)` / `trace_aer_event(...)` / `trace_non_standard_event(...)` | RAS tracepoint emitters | `Tracepoint::*` |

## Compatibility contract

REQ-1: MC layering: `mci->n_layers ∈ {1, 2, 3}`; each `mci->layers[i].type ∈ {EDAC_MC_LAYER_BRANCH, _CHANNEL, _SLOT, _CHIP_SELECT, _ALL_MEM}`; `csrows / dimms` arrays sized as Cartesian product of layer sizes.

REQ-2: Per-MC `mci->mtype_cap` advertises supported memory types (DDR, RDDR, DDR2..5, FB-DDR, RDIMM, LRDIMM, NVDIMM, HBM, HBM2, HBM3, WIO, WIO2); per-DIMM `dimm->mtype` selects active type.

REQ-3: Per-MC `mci->edac_ctl_cap` advertises supported ECC modes (NONE, RESERVED, PARITY, EC, SECDED, S4ECD4ED, S8ECD8ED, S16ECD16ED); per-DIMM `dimm->edac_mode` selects active mode.

REQ-4: Owner lock: `edac_mc_owner` ensures only one MC source (`apei/ghes` or one of the per-platform drivers like `i7core_edac`, `sb_edac`, `skx_base`, `i10nm_base`) is active at a time; attempts to register a second concurrent MC source return `-EBUSY`.

REQ-5: Operation states (`edac_op_state`): `POLL` (driver workqueue callback `mci->edac_check`), `NMI` (CMCI / MCE delivered NMI), `INT` (per-MC interrupt); `opstate_init` defaults invalid values to POLL.

REQ-6: Workqueue (`edac_workqueue`) initialized at subsystem init; poll interval per-MC computed from `edac_mc_poll_msec` (default 1000) bounded `[1, 1000]` ms.

REQ-7: Error normalization (`edac_mc_handle_error`): given `(type, mci, count, page_frame_number, offset_in_page, syndrome, top_layer, mid_layer, low_layer, msg)`: walk DIMM topology to locate matching `dimm_info`, compose `edac_raw_error_desc`, increment per-CE/UE counters, emit `mc_event` tracepoint, dev_warn-log per gating policy.

REQ-8: Panic gate: `edac_mc_panic_on_ue = 1` causes `panic("EDAC MC: ...")` on UE; default 0.

REQ-9: EDAC device generic counters: each `edac_device_block` has `ce_count` + `ue_count` (sysfs-readable, `_handle_ce` / `_handle_ue` bumps); per-instance + per-device aggregate counters maintained.

REQ-10: GHES bridge: each APEI-reported memory error converted into `edac_raw_error_desc` via `ghes_edac_report_mem_error` → `edac_raw_mc_handle_error`; provides a "ghes" MC instance when no per-platform driver is bound.

REQ-11: sysfs surface `/sys/devices/system/edac/{mc,cpu,pci}/` exposes per-MC `ce_count`, `ue_count`, `ce_noinfo_count`, `ue_noinfo_count`, `mc_name`, `seconds_since_reset`, `size_mb`, per-DIMM `dimm{label,location,size,mem_type,edac_mode,ce_count,ue_count}`, per-device class `seconds_since_reset` + per-block CE/UE.

REQ-12: Module parameter surface (`edac_mc_panic_on_ue`, `edac_mc_log_ue`, `edac_mc_log_ce`, `edac_mc_poll_msec`) writable via sysfs `/sys/module/edac_core/parameters/` and gated by CAP_SYS_ADMIN.

## Acceptance Criteria

- [ ] AC-1: On a SKX / ICX / SPR system, `skx_edac` / `i10nm_edac` binds and `/sys/devices/system/edac/mc/mc0/` enumerates with `n_layers >= 2`, per-DIMM label populated.
- [ ] AC-2: On an APEI/HEST-reporting platform, `ghes_edac` registers a "ghes" MC instance and routes firmware-delivered memory errors into `edac_raw_mc_handle_error`.
- [ ] AC-3: Synthetic CE injection via `/sys/kernel/debug/edac/mc0/inject_*` produces a `mc_event` tracepoint, increments `mc0/ce_count`, and is consumed by `rasdaemon`.
- [ ] AC-4: Synthetic UE injection with `edac_mc_panic_on_ue=1` calls `panic()`; with `=0`, emits `mc_event` UE + bumps `ue_count` without panic.
- [ ] AC-5: AMD MCE decoder (`mce_amd.c`) lifts per-CPU MCE bank events into `edac_raw_mc_handle_error` with matching DIMM label.
- [ ] AC-6: `edac_device_handle_ce` on an `aspeed_edac` / `altera_edac` L2 instance bumps the per-block CE counter and emits `mc_event` with `block_name`.
- [ ] AC-7: Second concurrent MC source registration is refused with `-EBUSY`; `edac_mc_owner` matches the active source string.
- [ ] AC-8: `edac_mc_poll_msec` writes outside `[1, 1000]` rejected by `edac_set_poll_msec` validator.

## Architecture

`MemCtlInfo` per-MC state (`drivers::edac::MemCtlInfo`):

```
struct MemCtlInfo {
  mc_idx: u32,
  pdev: &'static Device,
  mod_name: &'static str,
  ctl_name: &'static str,
  dev_name: KString,
  n_layers: u8,
  layers: [EdacMcLayer; MAX_LAYERS],
  csbased: bool,
  mtype_cap: MtypeCap,
  edac_ctl_cap: EdacCtlCap,
  edac_cap: EdacCtlCap,
  edac_check: Option<fn(&MemCtlInfo)>,    // poll-mode callback
  ctl_page_to_phys: Option<fn(...) -> phys_addr_t>,
  scrub_mode: ScrubMode,
  csrows: KVec<CsRowInfo>,
  nr_csrows: u32,
  dimms: KVec<DimmInfo>,
  tot_dimms: u32,
  error_desc: EdacRawErrorDesc,
  ce_count: AtomicU32,
  ue_count: AtomicU32,
  ce_noinfo_count: AtomicU32,
  ue_noinfo_count: AtomicU32,
  pvt_info: *mut c_void,
  workq: WorkStruct,
  poll_msec: u32,
}
```

`DeviceCtlInfo` per-device-class state:

```
struct DeviceCtlInfo {
  dev_idx: u32,
  dev: &'static Device,
  mod_name: &'static str,
  ctl_name: &'static str,
  edac_check: Option<fn(&DeviceCtlInfo)>,
  log_ce: bool,
  log_ue: bool,
  panic_on_ue: bool,
  poll_msec: u32,
  nr_instances: u32,
  instances: KVec<DeviceInstance>,        // e.g. per-CPU L2 caches
  blocks: KVec<DeviceBlock>,              // e.g. per-bank ECC slices
  pvt_info: *mut c_void,
  workq: WorkStruct,
}
```

Subsystem init (`edac_module.c`):
1. Create `edac_workqueue` (singlethread WQ with `WQ_MEM_RECLAIM`).
2. Initialize `edac_class` under `/sys/devices/system/edac/`.
3. Register debugfs root `/sys/kernel/debug/edac/`.
4. Register sysctl entries for module params.

MC registration (`edac_mc_alloc` → `edac_mc_add_mc`):
1. `edac_mc_alloc(num, n_layers, layers, sz_pvt)` allocates `mci` + `csrows` + `dimms` arrays as one chunk; populates per-DIMM `location[]` from layer indices.
2. Driver fills per-DIMM `mtype`, `dtype`, `edac_mode`, `nr_pages`, `grain`, `label`.
3. `edac_mc_add_mc(mci)`:
   - Acquire `mem_ctls_mutex`.
   - Verify `edac_mc_owner == NULL || edac_mc_owner == mod_name`.
   - Insert `mci` into `mc_devices` list.
   - Register sysfs nodes (`edac_create_sysfs_mci_device`).
   - Schedule workqueue if `edac_check` set.

Error reporting path (`edac_mc_handle_error`):
1. Construct `edac_raw_error_desc e` with `type`, `error_count`, `page_frame_number`, `offset_in_page`, `syndrome`, layer indices.
2. Resolve matching `dimm_info` via `edac_dimm_info_location` walk.
3. Concatenate per-DIMM labels separated by `OTHER_LABEL` (" or ") when error mask spans multiple DIMMs.
4. Atomic increment per-MC + per-DIMM `ce_count` or `ue_count`.
5. Emit `trace_mc_event(type, msg, label, error_count, mc_idx, top_layer, mid_layer, low_layer, address, grain, syndrome, driver_detail)`.
6. If `edac_mc_get_log_*()` set: `edac_mc_printk` with severity.
7. If `edac_mc_get_panic_on_ue() && type == HW_EVENT_ERR_UNCORRECTED`: `panic`.

Generic device path (`edac_device_handle_ce` / `_handle_ue`):
1. Walk `dev_ctl->instances[instance].blocks[block]`.
2. Bump `block.counters.ce_count` (or `.ue_count`).
3. Bump `dev_ctl.counters` aggregate.
4. Emit `trace_mc_event` with `block_name` instead of DIMM label.
5. If `dev_ctl->panic_on_ue && UE`: `panic`.

GHES bridge (`ghes_edac.c`):
1. `ghes_edac_register_mc(dev, num, name)` allocates a 1-DIMM, 1-channel "GHES" MC.
2. APEI receives memory error from firmware → calls `ghes_edac_report_mem_error(sev, mem_err)`.
3. Translates ACPI severity + physical address into `edac_raw_error_desc`, calls `edac_raw_mc_handle_error`.

## Hardening

- **`edac_mc_owner` lock** — only one MC source registered at a time; defense against double-counted errors or owner-id confusion.
- **Operation-state normalization** — `opstate_init` defaults invalid `edac_op_state` to POLL.
- **Poll-interval bounded** — `edac_set_poll_msec` validator refuses < 1 ms or > 1000 ms.
- **DIMM-label join via OTHER_LABEL** — multi-DIMM error groups produce a single trace entry; defense against fragmented event spam.
- **Per-MC ce_count / ue_count atomic** — defense against torn updates under concurrent CE/UE.
- **Panic gate default off** — explicit opt-in via sysfs / cmdline.
- **Debugfs error-injection requires CAP_SYS_RAWIO** — defense against unprivileged synthetic event injection.
- **GHES path uses APEI-validated payload** — refuses malformed CPER records.
- **Workqueue WQ_MEM_RECLAIM** — prevents OOM-induced poll starvation during memory pressure.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `mem_ctl_info`, `dimm_info`, `csrow_info`, `rank_info`, `edac_raw_error_desc`, `edac_device_ctl_info`, `edac_device_instance`, `edac_device_block`, `edac_pci_ctl_info`; reject userspace copies outside sysfs helpers.
- **PAX_KERNEXEC** — EDAC core (`edac_mc_alloc`, `edac_mc_add_mc`, `edac_mc_handle_error`, `edac_device_handle_ce`, `_handle_ue`, GHES bridge) in `__ro_after_init` text; per-driver `edac_check` poll callbacks in `__ro_after_init` text with W^X enforced.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across error-handling entry points (`edac_mc_handle_error`, `ghes_edac_report_mem_error`, NMI-mode MCE callbacks).
- **PAX_REFCOUNT** — saturating `refcount_t` on `mem_ctl_info`, `edac_device_ctl_info`, `edac_pci_ctl_info`; overflow trap defeats unbind-vs-error race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `mem_ctl_info` private state, `error_desc`, `dimm_info.label`, per-block counter snapshots so stale per-system topology and label data do not bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs read/write entering EDAC; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `mci->edac_check`, `mci->ctl_page_to_phys`, `dev_ctl->edac_check`, `edac_pci_ctl_info` operations vtables, `ghes_edac_report_mem_error` callback marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-MC / per-DIMM pointer disclosure behind CAP_SYSLOG; suppress `%p` in EDAC tracepoints.
- **GRKERNSEC_DMESG** — restrict CE/UE banners + per-DIMM label dumps to CAP_SYSLOG so attackers cannot fingerprint platform memory topology via dmesg.
- **`edac_mc` sysfs RO for non-root** — every per-DIMM `*_count`, `*_label`, `*_location`, `seconds_since_reset` file `mode = 0400` (root-readable) by default; refuse world-readable exposure of physical-DIMM identification.
- **Error-injection CAP_SYS_RAWIO** — debugfs `inject_section_type`, `inject_addr`, `inject_addr_mask`, `inject_syndrome`, `inject_*_count`, `mci_reset_counters` files restricted to CAP_SYS_RAWIO in init userns; reject from non-init userns.
- **Panic-on-uncorrectable gate** — `edac_mc_panic_on_ue` writes require CAP_SYS_BOOT; refuses concurrent change-while-handling-UE via owner-lock interlock.
- **Module-param mutation gate** — `edac_mc_log_ce`, `edac_mc_log_ue`, `edac_mc_poll_msec` writes via `/sys/module/edac_core/parameters/` require CAP_SYS_ADMIN; refuse from non-init userns.
- **GHES payload validation** — `ghes_edac_report_mem_error` validates `physical_addr` against `mci->ctl_page_to_phys` translatable range; refuses out-of-range firmware-provided addresses.
- **Tracepoint allowlist** — only `mc_event`, `aer_event`, `non_standard_event`, `arm_event`, `extlog_mem_event` registered; vendor-internal tracepoints behind separate gating.

Rationale: EDAC publishes per-DIMM physical identifiers (labels including socket / channel / slot / S/N), per-error physical addresses, syndromes, and per-DIMM error counts — a world-readable EDAC sysfs surface gives any local attacker fine-grained memory-controller fingerprinting + Rowhammer targeting telemetry. Error-injection debugfs hooks if reachable without CAP_SYS_RAWIO are a remote-DoS primitive (panic-on-UE). HIDESYM + DMESG gating on EDAC banners, sysfs-RO defaults on every per-DIMM file, debugfs RAWIO gating on injection, CAP_SYS_BOOT on panic-mode toggle, and kCFI on the per-driver `edac_check` poll callback turn EDAC from a passive observer into a mediated boundary that does not leak platform topology.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dimm_index_no_oob` | OOB | `edac_dimm_info_location` walks layer indices bounded by `mci->layers[i].size`. |
| `error_desc_label_no_oob` | OOB | per-DIMM `label` concatenation bounded by `EDAC_MAX_LABELS × (EDAC_MC_LABEL_LEN + len(OTHER_LABEL))`. |
| `ce_ue_count_no_overflow` | OVERFLOW | atomic CE/UE counters saturating (PAX_REFCOUNT semantics) rather than wrapping. |
| `owner_lock_no_double` | UNIQUENESS | `edac_mc_owner` enforces at most one active MC source. |

### Layer 2: TLA+

`models/edac/error_dispatch.tla` (parent-declared): models concurrent error sources (poll callback, NMI/MCE, GHES) under `edac_mc_panic_on_ue` toggle; proves panic occurs exactly when the first UE is observed with the flag set + that subsequent error reports do not race the panic.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `edac_mc_handle_error` precondition: `top_layer < mci->layers[0].size ∧ mid_layer < mci->layers[1].size ∧ low_layer < mci->layers[2].size` (or `-1` per missing layer). | `edac_mc` |
| `edac_mc_add_mc` post: `edac_mc_owner != NULL ∧ list_first(mc_devices) == mci`. | `edac_mc` |
| `edac_device_handle_ce` post: `dev_ctl->instances[i].blocks[b].counters.ce_count` incremented atomically. | `edac_device` |
| GHES bridge post: `edac_raw_mc_handle_error` invoked with `e.error_count >= 1 ∧ e.page_frame_number ∈ valid_pfn_range`. | `ghes_edac` |

### Layer 4: Verus/Creusot functional

End-to-end: a memory bank reports an ECC corrected error → per-platform MC driver computes `(top, mid, low)` layer indices + syndrome + page-frame-number → `edac_mc_handle_error` → `trace_mc_event` emits → `rasdaemon` reads tracepoint and persists; encoded as a Verus refinement from HW MCE bank decode to user-space-visible event.

## Out of Scope

- Per-platform MC drivers (skx_base, sb_edac, i10nm_base, amd64_edac, igen6_edac, ARM-SoC drivers): each gets a follow-on Tier-4 if needed.
- MCE-AMD decoder math (`mce_amd.c`): covered in `arch/x86/mce.md` Tier-3.
- APEI/HEST/CPER parsing internals: covered in `drivers/acpi/apei/00-overview.md` future Tier-3.
- CXL RAS lift into EDAC (covered in `cxl/00-overview.md`).
- 32-bit-only EDAC paths.
