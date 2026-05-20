# Tier-3: drivers/accel/habanalabs/ — Habana/Intel Gaudi/Goya AI accelerator (CS submit + MMU + firmware)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/accel/00-overview.md
upstream-paths:
  - drivers/accel/habanalabs/common/
  - drivers/accel/habanalabs/gaudi/
  - drivers/accel/habanalabs/gaudi2/
  - drivers/accel/habanalabs/goya/
  - include/uapi/drm/habanalabs_accel.h
-->

## Summary

The Habana Labs driver (originally `drivers/misc/habanalabs/`, migrated to `drivers/accel/` in 6.4) drives the Habana / Intel Gaudi family of AI training/inference accelerators — Goya (1st-gen inference), Gaudi (training), Gaudi2 (HBM2e + 96 TPCs + 24 MME), and the Gaudi2 revision variants (GAUDI2/2B/2C/2D distinguished by `pci_dev->revision`). It exposes a compute-accel char device `/dev/accel/accelN` with `DRIVER_COMPUTE_ACCEL` set, plus six Habana-specific DRM ioctls (`HL_INFO`, `HL_CB`, `HL_CS`, `HL_WAIT_CS`, `HL_MEMORY`, `HL_DEBUG`). The driver provides per-asic command-submission (CS) queues (external PCIe-DMA queues and internal TPC/MME/DMA queues), an asic-managed device MMU (driver-controlled 2-level page-tables for device virtual addressing — DRAM, SRAM, host-mapped, and DMA-coherent regions), an ARC firmware sandbox running on the device's ARM Cortex CPUs, secure-asic variants (Gaudi-SEC), hwmon temperature/power monitoring, decoder offload (Gaudi2), and a security module (`security.c`) that enforces signed-firmware mandatory load.

This Tier-3 covers `common/` (~25 files spanning device init, CS submit, memory management, MMU, firmware load, sysfs/debugfs, ioctl dispatch) and per-asic shims (`gaudi/`, `gaudi2/`, `goya/`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hl_device` | per-asic top-level state | `drivers::habanalabs::Device` |
| `struct hl_fpriv` | per-fd context (returned by `drm_file.driver_priv`) | `drivers::habanalabs::FilePriv` |
| `struct hl_ctx` | per-process accelerator context (MMU PASID-like) | `drivers::habanalabs::Context` |
| `struct hl_cs` | per-CS submission record | `drivers::habanalabs::CommandSubmit` |
| `struct hl_cb` (cb_pool) | command-buffer (CB) descriptor | `drivers::habanalabs::CommandBuffer` |
| `hl_device_open(ddev, file_priv)` | per-fd open: alloc `hl_fpriv` + `hl_ctx` | `Device::open` |
| `hl_device_release(ddev, file_priv)` | per-fd close: drain CS, free context | `Device::release` |
| `hl_cs_ioctl(ddev, data, file_priv)` | submit CS (parse user buf, validate, enqueue to HW queue) | `Device::cs_ioctl` |
| `hl_wait_ioctl` / `hl_info_ioctl` / `hl_cb_ioctl` / `hl_mem_ioctl` / `hl_debug_ioctl` | UAPI entry points | `Device::*_ioctl` |
| `hl_mmu_map_page(ctx, vaddr, paddr, page_size, ...)` / `hl_mmu_unmap_page(...)` | per-context MMU map/unmap | `MmuV2::map_page` / `_unmap_page` |
| `hl_fw_load_fw_to_device(...)` / `hl_fw_init_cpu(...)` | firmware load + ARC bring-up | `Firmware::load` / `Firmware::init_cpu` |
| `hl_hw_queue_send_cb_no_cmpl` / `hl_hw_queue_submit_bd` | per-queue PI write + doorbell | `HwQueue::submit` |
| `gaudi2_security_init` / `goya_security_init` | per-asic security register lock-down | `Security::init` |
| `hl_get_pll_info` / `hl_set_pll_freq` | clock control via firmware mailbox | `Firmware::pll_*` |
| `accel_open` / `DEFINE_DRM_ACCEL_FOPS` | inherited from accel-core | (see `00-overview.md`) |

## Compatibility contract

REQ-1: Per-asic detection by `pci_dev->device` + `revision`: `PCI_IDS_GOYA=0x0001` → ASIC_GOYA; `0x1000` → ASIC_GAUDI; `0x1010` → ASIC_GAUDI_SEC; `0x1020` → ASIC_GAUDI2/2B/2C/2D per rev. `PCI_VENDOR_ID_HABANALABS=0x1da3`.

REQ-2: Six UAPI ioctls (`HL_INFO`, `HL_CB`, `HL_CS`, `HL_WAIT_CS`, `HL_MEMORY`, `HL_DEBUG`) numbered relative to DRM base; ABI struct layout in `include/uapi/drm/habanalabs_accel.h` is stable.

REQ-3: Per-asic queue topology: Goya has 5 external PCIe-DMA queues + 1 CPU-PQ + internal MME/TPC queues; Gaudi has 8 external DMA queues across DMA_0/1/5 + CPU-PQ + many internal queues; Gaudi2 expands with HBM-DMA + EDMA + per-MME-engine queues.

REQ-4: Per-context MMU with 4 KB / 64 KB / 2 MB page sizes; per-context page-table allocated from `pgt_infos` pool; per-ASIC MMU version (v1 = Goya/Gaudi; v2 = Gaudi2; v2_hr = Gaudi2 host-resident page-tables).

REQ-5: CS submit validates per-CB legality via security registers (block-list of registers whose writes are denied from user CBs); per-asic `hl_asic_funcs.parse_cb` walks CB packets.

REQ-6: ARC firmware (per-asic `bin/habanalabs/`) signature-verified before load; un-signed FW rejected unless `hl_disable_secure_boot` debug knob set (CAP_SYS_RAWIO).

REQ-7: Reserved memory carve-outs in `_KMD_SRAM_RESERVED_*` (Goya 32 KB, Gaudi 128 B for driver scratch) — userspace cannot map into the reserved region.

REQ-8: Per-asic sync-object / monitor namespace: collective-wait reserves `GAUDI_FIRST_AVAILABLE_W_S_SYNC_OBJECT=144` SOBs + `..._MONITOR=72` monitors; sync-stream reserves further; userland-accessible space begins after.

REQ-9: Timestamp-registration buffer max `TS_MAX_ELEMENTS_NUM = 1<<20` (1 M entries / 8 MB) — bounded against memory exhaustion.

REQ-10: hwmon registers per-asic temperature + power + voltage sensors; sysfs `/sys/class/hwmon/hwmonX/` populated.

REQ-11: Debugfs (`debugfs/accel/<dev>/`) exposes `memory_scrub`, `dma_size`, `i2c_data`, `device_release_watchdog_timeout`, and per-asic registers; gated CAP_SYS_RAWIO.

REQ-12: Reset flow: `hl_device_reset` triggered on TPC-MEM-ECC, RAZWI, or watchdog; firmware-mediated soft-reset preferred over PCI-FLR.

## Acceptance Criteria

- [ ] AC-1: Load driver on Gaudi2 PCI device → `dmesg` shows `accel0: HL device initialized`.
- [ ] AC-2: `DRM_IOCTL_HL_INFO` returns device name, driver version, hw-version, dram-size, sram-size.
- [ ] AC-3: `HL_CB` allocates a CB; `HL_CS` submits + `HL_WAIT_CS` returns CS_STATUS_COMPLETED.
- [ ] AC-4: `HL_MEMORY` MAP_OP=ALLOC → DEVICE allocates DRAM; MAP_OP=MAP_DEVICE binds to per-context virtual addr; UNMAP releases; pte-walk confirms unmapping.
- [ ] AC-5: Multi-process: two processes each open `/dev/accel/accel0`, allocate per-context MMU PT, submit non-overlapping CS — neither sees the other's BOs.
- [ ] AC-6: Firmware signature failure (corrupted FW): driver refuses to enable accelerator; sysfs `status` reads `disabled`.
- [ ] AC-7: hwmon: `cat /sys/class/hwmon/hwmonX/temp1_input` returns valid temperature.
- [ ] AC-8: Reset test: trigger RAZWI → `hl_device_reset` → device returns to ready; pending CS error-completed.

## Architecture

`Device` (per-PCI bind):

```
struct Device {
  pdev: &PciDev,
  drm_dev: KBox<DrmDevice>,        // DRIVER_COMPUTE_ACCEL minor
  asic_type: AsicType,             // GOYA/GAUDI/GAUDI_SEC/GAUDI2/2B/2C/2D
  asic_funcs: &'static AsicFuncs,  // per-asic vtable (parse_cb, mmu_init, fw_load, …)
  asic_prop: AsicProp,             // DRAM size, SRAM base, queue count, MMU version
  hw_queues: KBox<[HwQueue]>,      // per-queue PI/CI rings + doorbell
  cs_mirror: Mutex<HashMap<u64, Arc<CommandSubmit>>>,
  cb_pool: KBox<CbPool>,
  mmu_priv: KBox<MmuPriv>,         // per-asic MMU state (v1/v2/v2_hr)
  pgt_infos: KBox<PgtPool>,
  fw: KBox<FirmwareState>,         // ARC firmware version, signatures, mailbox
  security: KBox<Security>,        // register block-list, signed-fw flag
  ctx_mgr: Mutex<HashMap<u32, Arc<Context>>>,
  reset_in_progress: AtomicBool,
  hwmon: KBox<HwmonState>,
  decoder: Option<KBox<DecoderState>>, // Gaudi2 H.264/H.265 decoder
  debugfs: KBox<DebugfsRoot>,
}

struct Context {
  refcount: Refcount,
  asid: u32,                       // per-context address-space id (HW-tagged)
  mmu_ctx: KBox<MmuCtx>,           // root pgd + DRAM/host VA ranges
  cs_seq: AtomicU64,
  hr_mmu_root: Option<PhysAddr>,   // host-resident page-table root (v2_hr)
  vmas: Mutex<RangeMap<DeviceVa, BoVma>>,
}
```

Per-fd open `Device::open(drm_file)`:
1. `hl_fpriv = KBox::new(FilePriv::default())`.
2. `hl_ctx = Context::alloc(this)` → allocate ASID, init per-context MMU root, initialize VMA map.
3. `drm_file.driver_priv = hl_fpriv`.

CS submit `Device::cs_ioctl`:
1. Copy `union hl_cs_args` from userland (PAX_UDEREF-checked).
2. Per-CS-chunk: copy chunk descriptors (cs-restore, cs-execute, cs-store) with bounded count.
3. For each `CB-handle + offset`: look up CB in `cb_pool` (refcount-incr); validate chunk fits within CB.
4. Per-asic `asic_funcs.parse_cb` walks CB packets, refusing writes to security-block-listed registers.
5. Build per-queue HW-descriptors; write into queue PI ring; doorbell write to kick HW.
6. Insert `hl_cs` into `cs_mirror` keyed by sequence; return seq to user.
7. `HL_WAIT_CS` polls `hl_cs.fence` (DRM sync-fence / wait-queue).

MMU map `MmuV2::map_page(ctx, vaddr, paddr, page_size, mmu_prop, …)`:
1. Validate vaddr ∈ ctx-owned VA range (DRAM-virt or host-virt).
2. Walk 4-level pgtable from `ctx.mmu_ctx.root`, allocating PT pages from `pgt_pool`.
3. Write leaf PTE with phys addr + valid + cache-attrs + access-perms.
4. Invalidate device-TLB via mailbox / direct register write (per-asic).
5. Track in `ctx.vmas` for unmap.

Firmware load `Firmware::load`:
1. `request_firmware("habanalabs/<asic>/<file>.bin", &pdev->dev)`.
2. Verify signature against embedded Habana public key (per-asic).
3. DMA-copy FW image into device SRAM/DRAM region.
4. Mailbox-write boot-fit version; ARC takes over from bootrom.
5. Wait for FW-ready handshake.

Per-asic security `Security::init` (Gaudi2 example):
1. Build block-list of MMIO offsets that user-CBs cannot target (PLL, power, security, MMU registers).
2. Program HW-protection (RANGE_PROTECTION_HBW_LO/HI) to enforce in hardware.
3. Set `security_enabled = true` — driver refuses to disable post-init.

## Hardening

- **Per-CS parse refuses unknown packets** — `parse_cb` returns `-EINVAL` on unrecognized opcode rather than passing to HW.
- **CB chunk count bounded** — per-CS chunks capped (`HL_MAX_JOBS_PER_CS`) to prevent OOM.
- **`cs_mirror` entries reference-counted** — concurrent close+wait safe.
- **MMU PT pages freed via RCU** — concurrent device DMA + host walk safe.
- **Per-asic register block-list immutable** — `__ro_after_init`.
- **CB lifetime tracked via refcount** — `HL_CB_DESTROY` blocked while CS using CB is in flight.
- **Reset watchdog** — per-CS deadline; on expiry, fence completes with error + device reset.
- **HW queue PI/CI bounded reads** — ring-index masked to queue-size at every dereference.
- **Decoder buffer bounds** — per-frame buffer size validated against asic max.
- **Signed-firmware default-on** — `hl_disable_secure_boot` requires CAP_SYS_RAWIO + module-param hint.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `hl_fpriv`, `hl_ctx`, `hl_cs`, `hl_cb`, MMU PT-info pool, and CS-args copy buffers; per-CS user-buf strictly bounded against `HL_MAX_JOBS_PER_CS` and per-CB size.
- **PAX_KERNEXEC** — Habana driver core in W^X kernel image; per-asic `asic_funcs` vtable, security block-list table, and firmware verification routines in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `hl_cs_ioctl`, `hl_mem_ioctl`, `hl_mmu_map_page`, and reset-handler entry points.
- **PAX_REFCOUNT** — saturating `refcount_t` on `hl_device`, `hl_ctx`, `hl_cb`, `hl_cs`, and `hl_fpriv` so the documented concurrent close+wait+CS race cannot underflow into UAF.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `hl_ctx`, MMU PT pages, CB DMA-coherent buffers, CS-args copy buffers, and ARC firmware mailbox scratch so stale device-VA mappings cannot bleed across context teardown.
- **PAX_UDEREF** — SMAP/PAN enforced on every Habana ioctl entry; per-CS chunk descriptors, manage-msg payloads, and MMU-map args copied via `copy_from_user` only, never raw user-pointer deref.
- **PAX_RAP / kCFI** — `drm_driver.ioctls`, per-asic `hl_asic_funcs` (`parse_cb`, `mmu_init`, `fw_load`, `hw_queues_init`, `restore_phase_topology`), and the firmware-mailbox callback vtable marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure behind CAP_SYSLOG; suppress `%p` of `hl_device`, `hl_ctx`, and per-CS pointers in Habana tracepoints + debugfs.
- **GRKERNSEC_DMESG** — restrict per-CS error, RAZWI, ECC, and firmware-load banners to CAP_SYSLOG so attackers cannot probe device-VA layouts via dmesg.
- **CS submit PAX_USERCOPY** — `union hl_cs_args` and per-chunk descriptors round-tripped through bounded copy helpers; CS-restore/execute/store size capped per ABI.
- **MMU map CAP_SYS_ADMIN for cross-context** — per-context MMU map for the calling fd's own VA range allowed; cross-context map (used by sync-stream collective-wait setup) requires CAP_SYS_ADMIN in the owning user-ns.
- **Firmware signature mandatory** — un-signed FW load refused; `hl_disable_secure_boot` cmdline / module-param requires CAP_SYS_RAWIO + audit-log entry; Gaudi-SEC variant rejects override entirely.
- **Debugfs CAP_SYS_RAWIO** — `debugfs/accel/<dev>/{memory_scrub, dma_size, i2c_data, *_reg_dump}` gated CAP_SYS_RAWIO; `device_release_watchdog_timeout` write-locked once a value is set.
- **ARC firmware sandbox** — per-asic security registers (range-protection HBW_LO/HI) programmed to fence ARC firmware out of host memory regions outside its scratch; refuse to disable post-init.
- **Per-CS deadline + reset** — CS watchdog completes with error rather than hanging; reset path zeroes per-context state before reuse.

Rationale: Habana Gaudi devices are bus-mastering compute accelerators with multi-gigabyte HBM and a privileged ARC firmware that can DMA into host memory. A relaxed `parse_cb` (letting user CBs poke PLL/MMU registers), an unsigned firmware load, a CS-mirror UAF on concurrent close, or a `hl_disable_secure_boot` accessible without CAP_SYS_RAWIO would silently demote the accelerator into a kernel-privileged DMA primitive. RAP/kCFI on the per-asic vtable, refcount-trapping on `hl_ctx`/`hl_cs`, PAX_MEMORY_SANITIZE on MMU PT pages, mandatory firmware signatures, and CAP_SYS_ADMIN on cross-context MMU map turn Habana into a structurally constrained compute endpoint rather than a privileged DMA backdoor.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-asic CS packet encoding (covered in per-asic Tier-4s if needed)
- ARC firmware internals (Habana-proprietary)
- Userland synapse-AI runtime
- Gaudi3 (post-baseline)
- Implementation code
