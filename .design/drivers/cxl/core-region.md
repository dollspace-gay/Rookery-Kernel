# Tier-3: drivers/cxl/core/{region,hdm}.c — HDM decoder programming, interleave configuration, region assembly

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/cxl/00-overview.md
upstream-paths:
  - drivers/cxl/core/region.c
  - drivers/cxl/core/hdm.c
  - drivers/cxl/core/region_pmem.c
  - drivers/cxl/core/region_dax.c
  - drivers/cxl/cxl.h
-->

## Summary

CXL Host-Managed Device Memory (HDM) is exposed to system software as a tree of decoders programmed in series along the topology: a **root decoder** (representing one ACPI CFMWS entry — a system-physical-address window with a target host-bridge list and an interleave specification), zero or more **switch decoders** (per CXL switch port), and one **endpoint decoder** per memdev that carves a Device Physical Address (DPA) extent out of the device into its slot in the SPA window. A **cxl_region** is the activated subset of decoders that together claim a contiguous SPA range and route it to a coherent set of memdev DPA ranges via the encoded Interleave Granularity (IG) and Interleave Ways (IW). Once committed, the region's SPA range may be onlined as system RAM (volatile / kmem path → `cxl/core/region_dax.c`) or attached as persistent memory through libnvdimm (`cxl/core/region_pmem.c`).

This Tier-3 covers `drivers/cxl/core/region.c` (~4100 lines: sysfs region attrs, decoder ordering constraints, commit/decommit, hotplug interactions, access_coordinates) and `drivers/cxl/core/hdm.c` (~1280 lines: HDM decoder enumeration + setup + commit + DVSEC range-regs emulation). Cross-ref `cxl/00-overview.md` for subsystem framing.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cxl_hdm` | per-port HDM decoder cap block | `drivers::cxl::Hdm` |
| `struct cxl_decoder` | base decoder (sysfs `decoderM.N`) | `drivers::cxl::Decoder` |
| `struct cxl_root_decoder` | root decoder over a CFMWS SPA window | `drivers::cxl::RootDecoder` |
| `struct cxl_switch_decoder` | per-switch-port decoder | `drivers::cxl::SwitchDecoder` |
| `struct cxl_endpoint_decoder` | per-memdev endpoint decoder | `drivers::cxl::EndpointDecoder` |
| `struct cxl_region` | activated region in SPA | `drivers::cxl::Region` |
| `struct cxl_region_params` | uuid, ig, iw, res, targets, state | `drivers::cxl::RegionParams` |
| `struct cxl_rwsem` | global region/dpa rwsems | `drivers::cxl::Rwsem` |
| `devm_cxl_setup_hdm(port, info)` | map HDM decoder regs for a port | `Hdm::setup` |
| `devm_cxl_enumerate_decoders(cxlhdm)` | iterate decoder array, attach to port | `Hdm::enumerate_decoders` |
| `parse_hdm_decoder_caps(cxlhdm)` | read CAP register (count, IW mask, IG mask) | `Hdm::parse_caps` |
| `cxl_decoder_add_locked(cxld)` / `cxl_decoder_autoremove(parent, cxld)` | sysfs register / devm undo | `Decoder::add` / `_autoremove` |
| `cxl_region_alloc(parent, mode, id)` / `_free(...)` | region object lifecycle | `Region::alloc` / `_free` |
| `attach_target(cxlr, decoder, pos)` | bind endpoint decoder slot N to region | `Region::attach_target` |
| `cxl_region_commit(cxlr)` / `cxl_region_decommit(cxlr)` | program HDM regs along topology | `Region::commit` / `_decommit` |
| `cxl_dpa_alloc(cxled, size)` / `cxl_dpa_free(cxled)` | reserve / release DPA extent | `EndpointDecoder::dpa_alloc` / `_free` |
| `eig_to_granularity(eig, &gran)` / `granularity_to_eig(gran, &eig)` | IG ↔ encoded-IG translate | `Decoder::eig_*` |
| `eiw_to_ways(eiw, &ways)` / `ways_to_eiw(ways, &eiw)` | IW ↔ encoded-IW translate | `Decoder::eiw_*` |
| `cxl_region_perf_data_calc(cxlr)` | per-region access_coordinates from CDAT | `Region::perf_data_calc` |
| `cxl_region_perf_attrs_callback(...)` | NUMA-node access_coords update notifier | `Region::perf_attrs_cb` |
| `cxl_region_iomem_release(...)` | release SPA iomem on decommit | `Region::iomem_release` |

## Compatibility contract

REQ-1: Decoder count + target count + IG mask + IW mask parsed from per-port HDM CAP register (CXL 2.0 §8.2.5.12); decoder array offsets `CXL_HDM_DECODER0_*_OFFSET(i) = 0x20 * i + base`.

REQ-2: Per CXL 3.0 §8.2.5.19.7: `CXL_DECODER_MIN_GRANULARITY = 256` bytes; encoded IG (eig) range 0..6 maps to granularity 256 B .. 16 KiB; encoded IW (eiw) range 0..4 = 1/2/4/8/16 ways, 8..10 = 3/6/12 ways.

REQ-3: Region configuration order strictly enforced: (1) interleave granularity, (2) interleave ways (size), (3) decoder targets — out-of-order writes from sysfs return `-EINVAL`.

REQ-4: Region states form a state machine: `CXL_CONFIG_IDLE → INTERLEAVE_GRANULARITY → INTERLEAVE_WAYS → ACTIVE → COMMIT → RESET`; transitions guarded by `cxl_rwsem.region` (rwsem write).

REQ-5: HDM decoder COMMIT writes `CXL_HDM_DECODER0_CTRL_COMMIT` and polls `_COMMITTED` (success) or `_COMMIT_ERROR`; lock bit (`_CTRL_LOCK`) blocks further programming once set by firmware/platform.

REQ-6: Endpoint DPA allocation uses per-device DPA resource tree (volatile + persistent partitions); per-endpoint-decoder `dpa_res` reserves an extent + skip range; allocations bounded by partition capacity from `GET_PARTITION_INFO`.

REQ-7: Persistent regions require UUID set + label-storage area provisioned via `SET_LSA` mailbox; volatile regions skip UUID.

REQ-8: Region commit walks the topology from endpoint up to root: program endpoint decoder, then each switch decoder, then root decoder programmed by firmware (root decoders are read-only on most platforms — CXL 3.x adds CFMWS-creatable cases).

REQ-9: Per-region access_coordinates (read/write bandwidth + latency) computed by combining CDAT DSLBIS values along the topology path; published to HMAT memory-tiering via `cxl_region_perf_attrs_callback`.

REQ-10: Region teardown invalidates CPU caches via `cpu_cache_has_invalidate_memregion` + `cpu_cache_invalidate_all` before SPA reuse to prevent stale-cache-line attacks on memory pool re-allocation.

REQ-11: DVSEC range-register emulation provided for endpoints without HDM decoder cap: `should_emulate_decoders` evaluates `info->mem_enabled` + DVSEC range regs to back-fill an "emulated" decoder.

REQ-12: Two global rwsems (`cxl_rwsem.region`, `cxl_rwsem.dpa`) coordinate region + DPA reservation across concurrent sysfs writers and topology hotplug events.

## Acceptance Criteria

- [ ] AC-1: `cxl create-region -d decoderN.0 -t pmem -w 2 -g 4096 -s 1G memX memY` produces a `regionN` device with state `commit` and an SPA range claimed.
- [ ] AC-2: Persistent region survives a memdev power cycle (label-area-based reassembly); volatile region disappears.
- [ ] AC-3: Misordered sysfs writes (`target_list` before `interleave_ways`) fail with `-EINVAL` and leave region state unchanged.
- [ ] AC-4: COMMIT against a locked HDM decoder (`_CTRL_LOCK` set by firmware) returns `-EBUSY`; no register modification occurs.
- [ ] AC-5: Region perf attrs (`access0/read_bandwidth`, `access0/read_latency`, etc.) populated when DSLBIS present in CDAT; missing → `-ENOENT` (file invisible).
- [ ] AC-6: Teardown of an onlined volatile region triggers `cpu_cache_invalidate_all` before SPA reuse (observable via tracepoint).
- [ ] AC-7: kselftest `tools/testing/selftests/cxl/` region create / commit / decommit sequence passes on `cxl_test` mocked topology.

## Architecture

`Hdm` per-port state lives at `drivers::cxl::Hdm`:

```
struct Hdm {
  port: Arc<Port>,
  regs: HdmDecoderRegs,           // memremap'd HDM decoder block
  decoder_count: u8,
  target_count: u8,
  interleave_mask: u32,           // bits 11_8 / 14_12 capability
  iw_cap_mask: u64,               // supported interleave-ways mask
}

struct Region {
  dev: Device,
  id: u32,
  mode: PartitionMode,             // RAM / PMEM
  type_: DeviceType,
  refcount: Refcount,
  params: RwLock<RegionParams>,    // uuid, ig, iw, res, targets[], state
  coord: [AccessCoordinate; 2],    // access0 (CPU side), access1 (CXL side)
  cxlr_pmem: Option<Arc<NdRegion>>, // libnvdimm hook for PMEM
}

struct RegionParams {
  uuid: Uuid,
  interleave_ways: u8,
  interleave_granularity: u16,
  res: Option<Resource>,           // SPA range
  state: ConfigState,
  targets: Vec<Arc<EndpointDecoder>>,
  nr_targets: u8,
}
```

HDM enumeration (`devm_cxl_setup_hdm` + `devm_cxl_enumerate_decoders`):
1. Map per-port HDM decoder block from `port->reg_map`.
2. `parse_hdm_decoder_caps()` reads CAP register: decoder count, target count, IW mask, IG mask.
3. For each decoder index N: read base_low/high, size_low/high, ctrl, target-list-low/high; classify root / switch / endpoint based on port type.
4. `cxl_root_decoder_alloc` / `_switch_decoder_alloc` / `_endpoint_decoder_alloc` allocates per-type object.
5. `cxl_decoder_add_locked` registers `decoderM.N` sysfs node under the port.

Region create (`create_pmem_region_store` / `create_ram_region_store` sysfs):
1. Acquire `cxl_rwsem.region` (write).
2. Allocate region id under root decoder's region_ida.
3. `cxl_region_alloc(parent_rd, mode, id)` builds `Region` in `IDLE` state.
4. Register sysfs node `regionN` under parent root decoder.

Region configuration (sysfs ordering):
1. `interleave_granularity_store(g)` — validate `g` is power-of-2 in `[256, 16384]`, encode via `granularity_to_eig`, state → `INTERLEAVE_GRANULARITY`.
2. `interleave_ways_store(w)` — validate `w` against `iw_cap_mask` of every topology port; allocate `targets[w]` slot array; state → `INTERLEAVE_WAYS`.
3. `target_list_store("memX,memY,...")` — per position: `attach_target(cxlr, endpoint_decoder, pos)`; verifies endpoint decoder belongs to a port path that reaches the parent root decoder; verifies DPA extent already allocated.
4. `size_store(sz)` — validate `sz % (ig * iw) == 0`; reserve SPA from root decoder's `res`; state → `ACTIVE`.
5. `commit_store(1)` — for each decoder along path (endpoint → switch → root): write base/size/ctrl/target-list to HDM regs; set COMMIT bit; poll COMMITTED; state → `COMMIT`.
6. `commit_store(0)` — DECOMMIT in reverse; invalidate CPU caches; release SPA; state → `RESET` then `ACTIVE`.

DPA allocation (`cxl_dpa_alloc`):
1. Acquire `cxl_rwsem.dpa` (write).
2. Within parent partition extent (volatile or persistent), find a free range of requested size via `cxlds->dpa_res`.
3. Insert a sub-resource on `cxled->dpa_res` (per-endpoint-decoder).
4. For PMEM mode: bump `cxled->skip` for any forced alignment gap.

Decoder COMMIT (per-decoder MMIO sequence):
1. Write `CXL_HDM_DECODER0_BASE_{LOW,HIGH}_OFFSET(i)` with SPA base.
2. Write `CXL_HDM_DECODER0_SIZE_{LOW,HIGH}_OFFSET(i)` with size.
3. Write `CXL_HDM_DECODER0_TL_{LOW,HIGH}(i)` with target-list (port id encoding).
4. Write `CXL_HDM_DECODER0_CTRL_OFFSET(i)` with IG | IW | HOSTONLY | COMMIT.
5. Poll ctrl until `_COMMITTED` (success) or `_COMMIT_ERROR` (-EIO).

Region perf data (`cxl_region_perf_data_calc`):
1. For each endpoint in region targets: look up DSMAS/DSLBIS from CDAT.
2. Sum latencies along topology path; aggregate bandwidth per IW.
3. Publish to `region->coord[CXL_ACCESS_LEVEL_*]`; emit notifier so memory-tiers updates NUMA-node attributes.

## Hardening

- **Interleave granularity bounded** — `granularity_to_eig` refuses non-power-of-2 or out-of-range (< 256 or > 16 KiB); defense against malformed sysfs input causing HDM-reg corruption.
- **Interleave ways validated against topology cap** — `ways_to_eiw` refuses ways outside `{1,2,3,4,6,8,12,16}` AND every port on path must advertise support in `iw_cap_mask`.
- **Decoder COMMIT polled with timeout** — refuses to advance state on `_COMMIT_ERROR`; defense against firmware lock-in producing inconsistent SPA mapping.
- **HDM decoder LOCK respected** — when `_CTRL_LOCK` set by firmware, all writes refused with `-EBUSY`; defense against firmware/OS arbitration race.
- **SPA range reserved via resource tree** — `request_resource` against parent root-decoder resource; defense against overlapping region claims.
- **DPA reservation bounded by partition** — endpoint decoder DPA extent strictly within `volatile_size` or `pmem_size` from `GET_PARTITION_INFO`.
- **Region state machine enforced** — out-of-order sysfs writes fail before MMIO; defense against ad-hoc reconfig under live traffic.
- **CPU cache invalidate before SPA reuse** — on decommit, `cpu_cache_invalidate_all` runs (when supported) to prevent stale cacheline disclosure across region reassembly.
- **Region rwsem global** — coordinates concurrent sysfs writers + topology hotplug + suspend.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `cxl_region`, `cxl_region_params`, `cxl_endpoint_decoder`, `cxl_root_decoder`, `cxl_switch_decoder`, HDM register shadow blocks; reject userspace copies outside the sysfs attr helpers.
- **PAX_KERNEXEC** — region + HDM code (`cxl_region_commit`, `cxl_region_decommit`, `attach_target`, `cxl_dpa_alloc`, `parse_hdm_decoder_caps`, decoder COMMIT polling) in `__ro_after_init` text with W^X enforced.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across sysfs region-create / commit / decommit / target_list_store entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `cxl_region`, `cxl_decoder`, `cxl_endpoint_decoder`, `cxl_root_decoder`; overflow trap defeats hotplug/teardown races during commit.
- **PAX_MEMORY_SANITIZE** — zero-on-free for HDM register shadow buffers, region params (incl. UUID + target list), DPA reservation records so stale interleave configs and persistent identifiers cannot bleed into a reused region slot.
- **PAX_UDEREF** — SMAP/PAN enforced on every region sysfs store; reject user-pointer deref outside the canonical sysfs helpers.
- **PAX_RAP / kCFI** — region/decoder ops vtables, `cxl_region_perf_attrs_callback`, hotplug notifier callbacks marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-region/per-decoder pointer disclosure behind CAP_SYSLOG; suppress `%p` in HDM tracepoints.
- **GRKERNSEC_DMESG** — restrict `_COMMIT_ERROR`, decoder-lock, DPA-overlap, target-validation banners to CAP_SYSLOG so attackers cannot probe topology via dmesg.
- **CAP_SYS_ADMIN strict on HDM decoder write** — every region sysfs `_store` writer requires CAP_SYS_ADMIN in the owning user namespace; refuse from non-init userns.
- **PAX_USERCOPY hard on interleave-config** — `interleave_granularity` / `interleave_ways` / `target_list` / `size` / `commit` sysfs stores parse user input with bounds-checked helpers only; refuse `simple_strtoul`-style consumption.
- **Region-create lock** — single global `cxl_rwsem.region` (write) held across the entire IDLE→COMMIT sequence; defense against TOCTOU between target_list validation and COMMIT.
- **Decoder LOCK is honored** — firmware/platform-set lock bit causes immediate `-EBUSY`; refuse to override even with override-cmdline.
- **SPA exposure audit** — every successful COMMIT emits a structured audit record (root, ig, iw, targets, SPA base+size) gated to CAP_AUDIT_WRITE consumers; defense against silent topology reconfiguration.

Rationale: HDM decoders are the kernel's authoritative routing primitive between system physical addresses and CXL device DPA — a misprogrammed decoder hands one VM access to another's memory or aliases two regions onto the same SPA window. A refcount underflow on `cxl_endpoint_decoder` during commit, a relaxed interleave-ways check, a missing CPU-cache invalidate on decommit, or a sysfs-store TOCTOU around target_list will produce an exploitable memory-aliasing primitive. CAP_SYS_ADMIN on every store, region-rwsem coverage across COMMIT, decoder-LOCK respect, refcount saturation, and PAX_USERCOPY on the interleave-config path turn region assembly from "trust the sysfs writer" into a mediated, audited operation.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ig_no_oob` | OOB | `eig_to_granularity(eig)` total only for `eig ≤ 6`; invalid eig → `-EINVAL`. |
| `iw_no_oob` | OOB | `eiw_to_ways(eiw)` total only for `eiw ∈ {0..4, 8..10}`; invalid → `-EINVAL`. |
| `decoder_array_no_oob` | OOB | per-port decoder index < `decoder_count`; MMIO offsets computed via `CXL_HDM_DECODER0_*_OFFSET(i)` bounded by mapped region. |
| `region_refcount_no_uaf` | UAF | region object outlives all attached endpoint decoders + nd_region bridge. |

### Layer 2: TLA+

`models/cxl/region_state_machine.tla` (parent-declared): models `IDLE → IG → IW → ACTIVE → COMMIT → RESET` with concurrent sysfs writers + endpoint hotplug; proves no commit succeeds with `targets[i] == NULL` for any `i < nr_targets`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Region::commit` precondition: `state == ACTIVE ∧ ∀i. targets[i].dpa_res ⊆ partition_extent`. | `core::region` |
| `Decoder::commit` postcondition (success): MMIO ctrl register reads back with `_COMMITTED` set; failure path leaves no partial decoder programmed. | `core::hdm` |
| `Region::decommit` postcondition: SPA range released from resource tree AND cache-invalidate executed before next region create reuses it. | `core::region` |
| `attach_target` precondition: endpoint decoder reachable from parent root via dport path; refuse otherwise. | `core::region` |

### Layer 4: Verus/Creusot functional

Region lifecycle: sysfs `create_pmem_region` → IG store → IW store → target_list store → size store → commit store → region claims SPA range with deterministic DPA→SPA mapping derivable from (ig, iw, target_list); encoded as a Verus refinement from sysfs operation sequence to HDM-register state.

## Out of Scope

- libnvdimm namespace activation atop PMEM region (covered in `nvdimm/libnvdimm.md`)
- DAX onlining of RAM region (covered in `dax/dax-core.md`)
- CXL FM-API region create over fabric-manager interface
- CXL switch-fabric routing tables
- CXL 3.0 Dynamic Capacity Device (DCD) extent management
