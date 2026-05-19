# Tier-3: drivers/fpga/{fpga-mgr,fpga-bridge,fpga-region}.c — FPGA manager class (programming, partial reconfiguration, region attach)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/fpga/fpga-mgr.c
  - drivers/fpga/fpga-bridge.c
  - drivers/fpga/fpga-region.c
  - drivers/fpga/of-fpga-region.c
  - include/linux/fpga/fpga-mgr.h
  - include/linux/fpga/fpga-bridge.h
  - include/linux/fpga/fpga-region.h
-->

## Summary

The FPGA subsystem provides three composable classes:

- **fpga_manager**: per-FPGA programming engine — accepts a bitstream (full or partial), drives the per-vendor configuration interface (Altera CvP / PR-IP / SoCFPGA / Xilinx ICAP / SelectMAP / Lattice sysCONFIG / Microchip / Versal), exposes a state-machine (`write_init` → `write` / `write_sg` → `write_complete`).
- **fpga_bridge**: per-bridge gate that isolates the soft-IP region from the static AXI fabric during reprogramming; enable/disable callbacks fence transactions so partial reconfiguration cannot corrupt the static side.
- **fpga_region**: composite descriptor binding a manager + a list of bridges + an `fpga_image_info` (bitstream + flags); `fpga_region_program_fpga` drives the entire flow (disable bridges → manager load → enable bridges).

This Tier-3 covers `fpga-mgr.c` (~1010 lines: manager registration, `fpga_mgr_load` state machine, sysfs class), `fpga-bridge.c` (~446 lines: bridge registration, enable/disable, list helpers), and `fpga-region.c` (~322 lines: region orchestration). It excludes per-vendor managers (each gets its own future Tier-3) and the OPAE DFL framework (orthogonal subsystem).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct fpga_manager` | per-FPGA manager device | `drivers::fpga::Manager` |
| `struct fpga_manager_ops` | vendor-driver vtable | `drivers::fpga::ManagerOps` |
| `struct fpga_manager_info` | registration aggregate (name, ops, parent, priv, compat_id) | `drivers::fpga::ManagerInfo` |
| `struct fpga_image_info` | per-load bitstream descriptor (data ptr OR sg OR firmware-name + flags + timeout) | `drivers::fpga::ImageInfo` |
| `struct fpga_compat_id` | image vs region compatibility uuid pair | `drivers::fpga::CompatId` |
| `struct fpga_bridge` / `fpga_bridge_ops` | per-bridge device + vtable | `drivers::fpga::Bridge` |
| `struct fpga_region` / `fpga_region_info` | per-region descriptor + registration aggregate | `drivers::fpga::Region` |
| `fpga_mgr_load(mgr, info)` | top-level programming entry: status → write_init → write/write_sg → write_complete | `Manager::load` |
| `fpga_mgr_lock(mgr)` / `_unlock(mgr)` | manager exclusive lock for in-progress reprogramming | `Manager::lock` |
| `fpga_image_info_alloc(dev)` / `_free(info)` | per-load info alloc/free | `ImageInfo::alloc` |
| `fpga_mgr_get(dev)` / `of_fpga_mgr_get(np)` / `fpga_mgr_put(mgr)` | consumer manager lookup | `Manager::get` |
| `__fpga_mgr_register(parent, name, ops, priv)` / `_register_full(parent, info)` / `fpga_mgr_unregister(mgr)` | manager class registration | `Manager::register` |
| `fpga_bridge_enable(bridge)` / `_disable(bridge)` | per-bridge gate | `Bridge::enable` / `_disable` |
| `fpga_bridges_enable(list)` / `_disable(list)` / `_put(list)` | list-of-bridges helpers | `Bridge::list_enable` |
| `of_fpga_bridge_get(np, info)` / `fpga_bridge_get(dev, info)` / `_get_to_list(...)` | bridge lookup | `Bridge::get` |
| `fpga_region_program_fpga(region)` | end-to-end region program: lock mgr, disable bridges, mgr load, enable bridges | `Region::program_fpga` |
| `__fpga_region_register_full(parent, info)` / `fpga_region_unregister(region)` | region class registration | `Region::register` |
| `fpga_region_class_find(start, data, match)` | global region iteration | `Region::class_find` |

## Compatibility contract

REQ-1: Manager state machine: `fpga_mgr_load(mgr, info)` invokes (a) `ops->state(mgr)` → must report `FPGA_MGR_STATE_OPERATING` or known transient; (b) `ops->write_init(mgr, info, buf, count)` (preamble, header parse, FPGA in PR mode); (c) `ops->write(mgr, buf, count)` or `ops->write_sg(mgr, sgt)`; (d) `ops->write_complete(mgr, info)` (CRC verify, DONE-bit wait, exit-PR mode).

REQ-2: Per-load image info fields: `buf` + `count` OR `sgt` OR `firmware_name` (mutually exclusive); `flags` carries `FPGA_MGR_PARTIAL_RECONFIG`, `FPGA_MGR_EXTERNAL_CONFIG`, `FPGA_MGR_ENCRYPTED_BITSTREAM`, `FPGA_MGR_BITSTREAM_LSB_FIRST`, `FPGA_MGR_COMPRESSED_BITSTREAM`; `region_id` and `header_size` per vendor.

REQ-3: Manager class: `/sys/class/fpga_manager/fpga%d/` with attributes `name`, `state` (per `enum fpga_mgr_states`), `status` (vendor bitstream-error flags).

REQ-4: Bridge: `fpga_bridge_enable(b)` invokes `ops->enable_set(b, true)`; `_disable` opposite; `ops->enable_show(b)` reads HW; per-bridge mutex `mutex` serializes.

REQ-5: Bridge list semantics: `fpga_bridges_disable(list)` walks list disabling each; `_enable(list)` walks in reverse enabling; `_put(list)` releases per-bridge refcount + clears list.

REQ-6: Region program: `fpga_region_program_fpga(region)` (a) takes per-region mutex; (b) `fpga_mgr_lock(region->mgr)`; (c) if `region->get_bridges`: populate `region->bridge_list`; (d) `fpga_bridges_disable(&region->bridge_list)`; (e) `fpga_mgr_load(region->mgr, region->info)`; (f) `fpga_bridges_enable(...)`; (g) unlock manager + region.

REQ-7: Compat-id check: `region->compat_id` (`{uuid_hi, uuid_lo}`) must match `info->image_id` (set by image-supplier driver) before program; mismatch returns `-EINVAL`.

REQ-8: DT-driven flow (`of-fpga-region.c`): on overlay apply, the region driver parses overlay's `firmware-name` + `partial-fpga-config` properties, builds `fpga_image_info`, calls `fpga_region_program_fpga`.

REQ-9: firmware-loader integration: if `info->firmware_name` set, framework calls `request_firmware(&fw, name, mgr->dev.parent)`; `fw->data` is fed to `write_init` + `write`.

REQ-10: Per-write chunk size: `mgr->mops->initial_header_size` and per-write `mgr->mops->align` constrain caller; framework feeds first header chunk via `write_init`, balance via `write` / `write_sg`.

REQ-11: Concurrency: per-manager `fpga_mgr_lock` mutex held across an entire load; concurrent loads serialized; bridge-list per-region locked similarly.

REQ-12: Hot-unregister: `fpga_mgr_unregister(mgr)` blocks until any in-progress load finishes; subsequent loads fail with `-ENODEV`.

## Acceptance Criteria

- [ ] AC-1: SoCFPGA / Cyclone V dev board: bootloader-deferred FPGA programming via `fpga_region` with overlay carrying `firmware-name = "soc.rbf"` succeeds; `fpga0/state` reads `operating`.
- [ ] AC-2: Xilinx Zynq full bitstream load via `zynq-fpga` returns success + DONE bit observable; AXI fabric to PL functional.
- [ ] AC-3: Partial reconfiguration on Stratix 10 / Versal: `FPGA_MGR_PARTIAL_RECONFIG` flag load succeeds without disturbing static region; bridge disable/enable observable.
- [ ] AC-4: Compat-id mismatch refuses load: `region->compat_id` differs from `info->image_id` → `-EINVAL`; no HW writes attempted.
- [ ] AC-5: Manager-lock contention: two concurrent `fpga_mgr_load` calls serialize; second sees consistent state after first completes.
- [ ] AC-6: Hot-unregister mid-load: `fpga_mgr_unregister` waits for in-flight load to finish; subsequent `fpga_mgr_load` returns `-ENODEV`.
- [ ] AC-7: KUnit `drivers/fpga/tests/` passes.
- [ ] AC-8: of-fpga-region overlay-apply path: device-tree overlay containing `firmware-name` + `external-fpga-config` triggers reprogram + driver-bind on children.

## Architecture

`Manager` + `Bridge` + `Region` layout:

```
struct Manager {
  name: KString,
  dev: Device,
  mops: &'static ManagerOps,
  mops_owner: Module,
  ref_mutex: Mutex<()>,
  state: AtomicEnum<MgrState>,
  status: AtomicU64,
  compat_id: Option<KBox<CompatId>>,
  priv: *mut c_void,
}

trait ManagerOps {
  fn initial_header_size() -> usize;
  fn skip_header(&self, buf: &[u8]) -> Option<usize>;
  fn state(mgr: &Manager) -> MgrState;
  fn status(mgr: &Manager) -> u64;
  fn parse_header(mgr: &Manager, info: &mut ImageInfo, buf: &[u8]) -> Result<()>;
  fn write_init(mgr: &Manager, info: &ImageInfo, buf: &[u8], count: usize) -> Result<()>;
  fn write(mgr: &Manager, buf: &[u8], count: usize) -> Result<()>;
  fn write_sg(mgr: &Manager, sgt: &SgTable) -> Result<()>;
  fn write_complete(mgr: &Manager, info: &ImageInfo) -> Result<()>;
  fn fpga_remove(mgr: &Manager);
  fn groups: &'static [&'static AttributeGroup];
}

struct Bridge {
  name: KString,
  dev: Device,
  mutex: Mutex<()>,
  br_ops: &'static BridgeOps,
  info: KBox<BridgeInfo>,
  node: ListHead,                        // when on a region's bridge_list
  priv: *mut c_void,
}

struct Region {
  dev: Device,
  mutex: Mutex<()>,
  bridge_list: ListHead,
  mgr: Arc<Manager>,
  info: Option<KBox<ImageInfo>>,
  compat_id: KBox<CompatId>,
  priv: *mut c_void,
  get_bridges: Option<fn(&Region) -> Result<()>>,
}

struct ImageInfo {
  flags: u32,
  enable_timeout_us: u32,
  disable_timeout_us: u32,
  config_complete_timeout_us: u32,
  firmware_name: Option<KString>,
  sgt: Option<&SgTable>,
  buf: Option<*const u8>,
  count: usize,
  region_id: i32,
  dev: Option<Arc<Device>>,
  header_size: usize,
  data_size: usize,
}
```

Manager load `fpga_mgr_load(mgr, info)`:
1. `fpga_mgr_lock(mgr)` exclusive mutex.
2. If `info->firmware_name`: `request_firmware(&fw, info->firmware_name, mgr->dev.parent)`; set `info->buf = fw->data; info->count = fw->size`.
3. If `mops->parse_header`: invoke on `info->buf[..initial_header_size]`; populate `info->{header_size, data_size}`.
4. Compat-id check against region (when invoked through region).
5. State check: `mops->state(mgr)` must be `FPGA_MGR_STATE_OPERATING` or `_POWER_UP` or known transient; `_UNKNOWN` → fail.
6. `mops->write_init(mgr, info, info->buf, info->header_size)` — vendor preamble, FPGA enters PR mode.
7. `mops->write(mgr, info->buf + header_size, info->data_size)` OR `mops->write_sg(mgr, info->sgt)`.
8. `mops->write_complete(mgr, info)` — CRC verify, DONE wait, exit PR mode; per-`config_complete_timeout_us` cap.
9. Release firmware; `fpga_mgr_unlock(mgr)`.

Bridge enable/disable:
- `fpga_bridge_disable(b)`: `mutex_lock(&b->mutex); b->br_ops->enable_set(b, false); mutex_unlock`.
- `_enable(b)`: opposite.
- List helpers walk under each per-bridge mutex; per-region mutex serializes the whole list operation.

Region program `fpga_region_program_fpga(region)`:
1. `mutex_lock(&region->mutex)`.
2. `fpga_mgr_lock(region->mgr)`.
3. If `region->get_bridges`: invoke to populate `bridge_list` from `region->info` or DT.
4. `fpga_bridges_disable(&region->bridge_list)`.
5. `fpga_mgr_load(region->mgr, region->info)`.
6. `fpga_bridges_enable(&region->bridge_list)`.
7. `fpga_bridges_put(&region->bridge_list)`.
8. Release locks; on any failure, attempt to re-enable bridges + unlock.

DT-driven (`of-fpga-region.c`):
1. Overlay apply → notifier → `of_fpga_region_notify_pre_apply` invoked.
2. Build `fpga_image_info` from overlay's `firmware-name`, `external-fpga-config`, `partial-fpga-config`, `encrypted-fpga-config`, `region-unfreeze-timeout-us`, `region-freeze-timeout-us`, `config-complete-timeout-us` props.
3. Call `fpga_region_program_fpga`.
4. On post-apply, driver-bind children present in the overlay.

Sysfs: `state` shows current `fpga_mgr_states` enum; `status` shows last-known per-vendor flags.

## Hardening

- **Per-manager exclusive lock** — `fpga_mgr_lock` is the only legal entry to a programming sequence; concurrent loads serialize.
- **State machine strictly checked** — invalid `write` before `write_init` or after `write_complete` returns `-EINVAL`.
- **Per-operation timeouts** — `enable_timeout_us`, `disable_timeout_us`, `config_complete_timeout_us` cap bridge-enable + DONE-bit polling.
- **Bridges fenced before write** — partial reconfig cannot reach static fabric mid-write because all bridges report disabled.
- **Compat-id verified** — region rejects image with mismatched `image_id` vs `region.compat_id`.
- **`request_firmware` parented to manager's parent device** — defense against arbitrary firmware-name traversal.
- **Hot-unregister blocks in-flight load** — `fpga_mgr_unregister` waits on `ref_mutex` then marks state UNKNOWN.
- **Per-region mutex** — defense against concurrent overlay-apply on same region.
- **sg-table per-page validated** — `mops->write_sg` passed a kernel-owned sgt; userland-supplied DMA buffers refused.
- **Module pin** — `mops_owner` (`THIS_MODULE`) bumped on manager get; provider driver pinned across load.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `fpga_manager`, `fpga_bridge`, `fpga_region`, `fpga_image_info`, `fpga_compat_id`; reject user-supplied bitstream copy outside the firmware-loader path (which is itself path-vetted).
- **PAX_KERNEXEC** — fpga-mgr core, bridge core, region core, and per-vendor manager drivers in W^X kernel text; `fpga_manager_ops`, `fpga_bridge_ops`, region helper-ops marked `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset on `fpga_mgr_load`, `fpga_region_program_fpga`, `fpga_bridge_enable/_disable`, and overlay-apply notifier entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on manager class device, bridge class device, region class device, and image-info refcount; defense against teardown-vs-load races that previously underflowed kref.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `fpga_image_info` (incl. buf pointer copy), per-vendor manager private blobs, bridge private state, and any DMA-staging buffers so a prior bitstream cannot bleed into a fresh load.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs entry; reject user-pointer deref outside canonical `kobj_attribute` show/store helpers and the firmware-loader API.
- **PAX_RAP / kCFI** — `fpga_manager_ops` (`write_init`, `write`, `write_sg`, `write_complete`, `state`, `status`, `parse_header`, `fpga_remove`), `fpga_bridge_ops` (`enable_set`, `enable_show`, `fpga_bridge_remove`), and region helper-ops marked `__ro_after_init` and dispatched via kCFI-typed indirect calls.
- **GRKERNSEC_HIDESYM** — gate disclosure of per-manager `priv` pointer, per-bridge MMIO base, and per-region `priv` pointer in debugfs / `/proc/iomem` behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict per-load banners ("programming FPGA …", "CRC error", "DONE not set") to CAP_SYSLOG so attackers cannot probe bitstream provenance via dmesg.
- **Bitstream signature mandatory** — when CONFIG_FPGA_REGION_SECURITY enforced, every image must carry a vendor signature verifiable via the platform keyring (or a `secondary_trusted_keys`); reject unsigned images with `-EKEYREJECTED`.
- **`fpga_mgr_ops` kCFI** — vtable swap of `write` / `write_complete` defeated by kCFI; per-call hash check on indirect dispatch.
- **Partial-reconfig CAP_SYS_RAWIO** — every entry to `fpga_region_program_fpga` and direct `fpga_mgr_load` from userspace surfaces gated by CAP_SYS_RAWIO in the user namespace.
- **Region attach refcounted** — `region->mgr` + `region->bridge_list` entries hold refcounts; refuse `fpga_mgr_unregister` or `fpga_bridge_unregister` while any region references them.
- **Write-init / complete state machine** — fpga-mgr core refuses `write` without prior `write_init`; refuses second `write_complete`; refuses `write_sg` if vendor lacks the op; defense against bypass that programs FPGA from a partially-validated state.
- **Compat-id strict** — refuse load if `info->image_id` zeroed unless `FPGA_MGR_EXTERNAL_CONFIG` flag explicitly asserts caller-managed compatibility; reject overlay path without compat-id.
- **Firmware-name path policed** — `request_firmware` path strictly under `/lib/firmware/`; reject paths containing `..` or absolute non-canonical prefixes.

Rationale: an FPGA bitstream is functionally equivalent to executable code — it programs gates that talk to the kernel's AXI/PCIe fabric, can mint forged DMA masters, mount confused-deputy attacks, or exfiltrate secrets from shared on-die SRAM. Without signature enforcement, an unprivileged write to `firmware-name` plus an overlay-apply lets an attacker load arbitrary HDL. RAP/kCFI on `fpga_manager_ops` + `fpga_bridge_ops`, CAP_SYS_RAWIO on partial-reconfig entry, refcount discipline across manager/bridge/region/image, mandatory signature on bitstreams, and a strictly enforced `write_init` → `write` → `write_complete` state machine turn the FPGA subsystem from "load any blob the user names" into a signed, fenced, audited programming pipeline.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-vendor manager drivers (`zynq-fpga.c`, `socfpga-a10.c`, `stratix10-soc.c`, `xilinx-spi.c`, `altera-pr-ip-core.c`, `microchip-spi.c`, `lattice-sysconfig.c`, `versal-fpga.c`, `ice40-spi.c`, `machxo2-spi.c`, `ts73xx-fpga.c`, `xilinx-selectmap.c`) — each gets a future Tier-3 if depth warranted; baseline grsec inherits.
- DFL / OPAE Accelerator Functional Unit framework (`dfl-*.c`) — separate Tier-3 doc.
- Altera CvP PCIe-configuration variant (`altera-cvp.c`) — separate Tier-3.
- Lattice / Microchip per-chip secure-update flows — separate Tier-3.
- 32-bit-only paths.
- Implementation code.
