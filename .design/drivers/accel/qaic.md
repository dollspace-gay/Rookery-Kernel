# Tier-3: drivers/accel/qaic/ — Qualcomm Cloud AI 100 accelerator (BO + MHI + SAHARA boot)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/accel/00-overview.md
upstream-paths:
  - drivers/accel/qaic/qaic_drv.c
  - drivers/accel/qaic/qaic_data.c
  - drivers/accel/qaic/qaic_control.c
  - drivers/accel/qaic/mhi_controller.c
  - drivers/accel/qaic/sahara.c
  - drivers/accel/qaic/qaic_ras.c
  - drivers/accel/qaic/qaic_ssr.c
  - drivers/accel/qaic/qaic_timesync.c
  - include/uapi/drm/qaic_accel.h
-->

## Summary

The QAIC driver (`drivers/accel/qaic/`) drives the Qualcomm Cloud AI 100 family of AI inference accelerators (AIC100, AIC200), exposed as a PCIe endpoint that boots from the host via the Qualcomm SAHARA bootloader protocol and then operates as an MHI (Modem Host Interface) endpoint. The driver presents a `DRIVER_GEM | DRIVER_COMPUTE_ACCEL` DRM minor with nine QAIC ioctls (`QAIC_MANAGE`, `QAIC_CREATE_BO`, `QAIC_MMAP_BO`, `QAIC_ATTACH_SLICE_BO`, `QAIC_EXECUTE_BO`, `QAIC_PARTIAL_EXECUTE_BO`, `QAIC_WAIT_BO`, `QAIC_PERF_STATS_BO`, `QAIC_DETACH_SLICE_BO`). Workloads are pre-compiled neural-network "Containers" that the host loads via a sideband transport (MHI QAIC_CONTROL channel) using transaction-typed messages (`QAIC_TRANS_*`), and runtime BOs hold input/output tensors that get sliced and DMA-streamed to the device. The driver also stands up MHI channels for SAHARA boot, DIAG, SSR (Sub-System Restart for crash recovery), QDSS trace, telemetry, IPCR, time-sync, and per-AIC RAS error reporting.

This Tier-3 covers `qaic_drv.c` (driver/PCI bind, DRM driver registration), `qaic_data.c` (BO + slice + execute), `qaic_control.c` (MANAGE ioctl + transaction protocol), `mhi_controller.c` (MHI channel definitions), `sahara.c` (host-side bootloader), `qaic_ssr.c`, `qaic_ras.c`, `qaic_timesync.c`, and `qaic_debugfs.c`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct qaic_device` | per-PCIe device state | `drivers::qaic::Device` |
| `struct qaic_drm_device` | per-DRM minor (one per partition) | `drivers::qaic::DrmDevice` |
| `struct qaic_user` (`qaic_drm_device.users`) | per-fd context | `drivers::qaic::User` |
| `struct qaic_bo` | per-BO state (GEM + DMA-streaming slice list) | `drivers::qaic::Bo` |
| `qaic_pci_probe` / `_remove` | PCIe bind / unbind | `Device::probe` / `_remove` |
| `qaic_open(drm_dev, drm_file)` / `_postclose(...)` | per-fd open/close | `DrmDevice::open` / `_postclose` |
| `qaic_manage_ioctl(drm_dev, data, file_priv)` | run transaction-typed message over MHI QAIC_CONTROL | `DrmDevice::manage_ioctl` |
| `qaic_create_bo_ioctl` / `_mmap_bo_ioctl` / `_attach_slice_bo_ioctl` / `_execute_bo_ioctl` / `_partial_execute_bo_ioctl` / `_wait_bo_ioctl` / `_perf_stats_bo_ioctl` / `_detach_slice_bo_ioctl` | per-BO UAPI ops | `DrmDevice::*_ioctl` |
| `qaic_mhi_register_controller(pdev, mhi_aic100_config)` | MHI controller init + channel allocation | `Device::mhi_register` |
| `sahara_register()` / `sahara_unregister()` | MHI QAIC_SAHARA channel client | `Sahara::register` / `_unregister` |
| `qaic_ssr_init` / `qaic_ssr_remove` | MHI QAIC_SSR client (sub-system restart) | `Ssr::init` / `_remove` |
| `qaic_ras_register` / `_unregister` | RAS error-event MHI client | `Ras::register` / `_unregister` |
| `qaic_timesync_init` / `_remove` | time-sync MHI client | `Timesync::init` / `_remove` |
| `qaic_destroy_drm_device` / `qaic_create_drm_device` | per-partition DRM lifecycle | `Device::create_drm` / `_destroy_drm` |
| `aic100_channels[]` / `aic200_channels[]` | MHI channel allowlists | `MhiConfig::AIC100` / `MhiConfig::AIC200` |
| `qaic_gem_prime_import` | dma-buf import path | `Bo::prime_import` |

## Compatibility contract

REQ-1: PCIe IDs `PCI_VENDOR_ID_QCOM` device IDs for AIC100 / AIC200 (variant tables); `DRIVER_GEM | DRIVER_COMPUTE_ACCEL` set.

REQ-2: Nine UAPI ioctls per `include/uapi/drm/qaic_accel.h`; ABI struct layouts (`qaic_manage_msg`, `qaic_create_bo`, `qaic_attach_slice`, `qaic_execute_bo`, `qaic_wait_bo`, `qaic_perf_stats_bo`) stable.

REQ-3: MANAGE message length capped at `QAIC_MANAGE_MAX_MSG_LENGTH = SZ_4K` including header + transaction count.

REQ-4: Transaction types per `QAIC_TRANS_*` (passthrough, dma_xfer, activate, deactivate, status, terminate, validate_partition); each has from-/to-USR and from-/to-DEV variants; cross-direction validation enforced.

REQ-5: Semaphore flags `QAIC_SEM_INSYNCFENCE`/`OUTSYNCFENCE` + commands `QAIC_SEM_{NOP,INIT,INC,DEC,WAIT_EQUAL,WAIT_GT_EQ,WAIT_GT_0}` form the per-BO sync-stream primitive.

REQ-6: MHI channels per `aic100_channels[]` / `aic200_channels[]`: LOOPBACK, SAHARA, DIAG, SSR, QDSS, CONTROL, LOGGING, STATUS, TELEMETRY, DEBUG, TIMESYNC, TIMESYNC_PERIODIC, IPCR.

REQ-7: SAHARA boot protocol per `SAHARA_*_CMD` opcodes (HELLO, READ_DATA, END_OF_IMAGE, DONE, RESET, MEM_DEBUG, CMD_READY, SWITCH_MODE, EXECUTE, MEM_DEBUG64, READ_DATA64, RESET_STATE, WRITE_DATA); host streams signed firmware image to device bootloader.

REQ-8: Per-partition DRM minor: AIC100 single partition (`QAIC_NO_PARTITION`) per `qaic_create_drm_device`; multi-partition reserved for AIC200 future.

REQ-9: Per-BO slice list: `QAIC_ATTACH_SLICE_BO` defines DMA-streaming pattern (multi-segment) that EXECUTE iterates; PARTIAL_EXECUTE allows trailing-slice partial submit.

REQ-10: Per-fd `qaic_user.qddev_lock` SRCU protects against partition destroy racing fd operations.

REQ-11: SSR client receives crash event on MHI QAIC_SSR channel → driver triggers partition teardown + DRM-device unregister + wait-for-MHI-restart.

REQ-12: RAS client streams device-error events on MHI QAIC_RAS channel; severity-classified + reported via `drm_dev_err`.

## Acceptance Criteria

- [ ] AC-1: Insert AIC100 PCIe card → `dmesg` shows MHI controller probe; SAHARA boot completes; `/dev/accel/accelN` appears.
- [ ] AC-2: `DRM_IOCTL_QAIC_MANAGE` with `QAIC_TRANS_ACTIVATE_FROM_USR` activates a partition; `STATUS_FROM_USR` returns ready.
- [ ] AC-3: `QAIC_CREATE_BO` (size + flags) creates a GEM object; `QAIC_MMAP_BO` returns offset; mmap → user gets buffer.
- [ ] AC-4: `QAIC_ATTACH_SLICE_BO` defines slice pattern; `QAIC_EXECUTE_BO` enqueues; `QAIC_WAIT_BO` returns completed.
- [ ] AC-5: `QAIC_PARTIAL_EXECUTE_BO` with `slices_count < attached_slices` executes prefix; subsequent partial executes resume.
- [ ] AC-6: `QAIC_DETACH_SLICE_BO` cleanly releases slice list; subsequent EXECUTE fails with `-EINVAL`.
- [ ] AC-7: SSR test: trigger device crash → SSR client tears down partition → MHI restart → driver re-registers DRM device.
- [ ] AC-8: Multi-process: two qaic_user contexts each allocate BOs, execute; each sees only its own BOs.
- [ ] AC-9: SAHARA signature failure: corrupted firmware → SAHARA negotiation aborts; driver enters fault state; partition not registered.

## Architecture

`Device` (per-PCIe bind):

```
struct Device {
  pdev: &PciDev,
  mhi_cntrl: KBox<MhiController>,    // MHI core controller
  mhi_channels: &'static [MhiChannelConfig], // aic100_channels or aic200_channels
  qddev: KBox<DrmDevice>,
  sahara: KBox<SaharaContext>,
  ssr: KBox<SsrContext>,
  ras: KBox<RasContext>,
  timesync: KBox<TimesyncContext>,
  dbcs: KBox<[DoorbellChannel]>,     // per-DBC: queue + interrupt + execute path
  fw_version: Atomic<u32>,
  state: Atomic<DeviceState>,         // BOOTING / READY / SSR / FAULT
  debugfs: KBox<DebugfsRoot>,
}

struct DrmDevice {
  drm: DrmDevice,
  qdev: &Device,
  partition_id: i32,
  users: Mutex<List<Arc<User>>>,
  users_mutex: Mutex<()>,
}

struct User {
  refcount: Refcount,
  qddev_lock: Srcu,                   // protects against partition destroy
  qddev: WeakRef<DrmDevice>,
  bos: Mutex<HashMap<u32, Arc<Bo>>>,
  perf_stats: PerfCounters,
}

struct Bo {
  gem_object: DrmGemObject,
  sgt: ScatterTable,                  // DMA-mapped SG list
  slices: Vec<BoSlice>,               // per-segment DMA-streaming descriptors
  semaphores: Vec<BoSemaphore>,
  fence: Option<DmaFence>,            // EXECUTE in-flight fence
  perf: PerfStats,                    // cycles / data-bytes / queue-depth
}
```

Probe flow `Device::probe(pdev)`:
1. PCI-enable + BAR map + DMA mask set.
2. `mhi_register_controller(pdev, &mhi_aic100_config)` → MHI core sets up event/cmd rings.
3. SAHARA channel opens; host receives `SAHARA_HELLO_CMD`; responds with `SAHARA_HELLO_RESP_CMD` (protocol-version + mode).
4. SAHARA streams firmware (`request_firmware("qcom/aic100/sbl.elf", ...)`) via `READ_DATA_CMD` / `READ_DATA64_CMD` until `END_OF_IMAGE_CMD`; signature verified by device bootloader (not host).
5. `SAHARA_DONE_CMD` → device transitions to MHI execution environment.
6. CONTROL channel opens → driver issues `validate_partition` MANAGE message → on success, `qaic_create_drm_device` registers DRM minor.
7. SSR / RAS / TIMESYNC clients register on their MHI channels.

MANAGE ioctl `DrmDevice::manage_ioctl(user, msg)`:
1. Copy `qaic_manage_msg` (len ≤ 4 KB) from userland.
2. Walk transactions (`qaic_manage_trans_hdr` → type + len); validate each fits in remaining buffer.
3. Per-transaction type:
   - `PASSTHROUGH_FROM_USR` → pack into MHI CONTROL payload as-is.
   - `DMA_XFER_FROM_USR` → resolve DMA descriptors against user's pinned pages.
   - `ACTIVATE_FROM_USR` → activate partition; record partition ID.
4. Send via MHI QAIC_CONTROL channel; await reply.
5. Walk reply transactions; copy `*_TO_USR` payloads back to userland (PAX_UDEREF-checked).

EXECUTE ioctl `DrmDevice::execute_bo_ioctl(user, args)`:
1. Look up BO by handle in `user.bos`; verify slices attached.
2. For each slice: build doorbell-channel descriptor (queue index + buffer offset + length + semaphore op).
3. Enqueue into per-DBC ring; ring doorbell.
4. Allocate `dma_fence` + record in `bo.fence`.
5. Return handle so userspace can `WAIT_BO`.

SAHARA boot `Sahara::handle_packet`:
1. Per-packet opcode dispatch.
2. `READ_DATA{,64}_CMD(offset, len)`: read `len` bytes at `offset` from current firmware blob; respond with raw bytes (zero-copy via MHI scatter-DMA).
3. `END_OF_IMAGE_CMD(status)`: if status != SUCCESS, abort + signal fault.
4. `MEM_DEBUG{,64}_CMD`: device requests host-RAM dump for crash analysis; bounded by exported MemDebug region.

SSR `Ssr::handle_event(event)`:
1. On `SSR_BEFORE_POWERUP` / `SSR_AFTER_POWERDOWN`: drain CONTROL channel + tear down DBCs + unregister DRM device for affected partition.
2. On `SSR_AFTER_POWERUP`: re-register DRM device once SAHARA + CONTROL handshake succeed.

## Hardening

- **MANAGE msg ≤ 4 KB** — fixed cap on copy_from_user.
- **Per-transaction length validated** — refuse trans whose `len` > remaining buffer.
- **MHI channel allowlist** — only the `aic100_channels[]`/`aic200_channels[]` entries are wired; unknown channel-id from device rejected by MHI core.
- **Per-user SRCU** — partition destroy waits for outstanding fd ops; no UAF on race.
- **Per-BO refcount via GEM** — close+execute race safe.
- **Slice attach validates DMA segment sums** — refuse attach where sum-of-slice-lengths > BO size.
- **PARTIAL_EXECUTE bounded** — `slices_count` validated against attached slices.
- **SSR teardown drains** — pending fences error-completed.
- **Time-sync periodic message bounded** — channel rate-limited.
- **Debugfs read-only** for FW version, partition state; write-paths CAP_SYS_RAWIO.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `qaic_device`, `qaic_user`, `qaic_bo`, `qaic_manage_msg` scratch, per-DBC ring elements, and slice descriptors; MANAGE-msg copy strictly bounded against `QAIC_MANAGE_MAX_MSG_LENGTH`.
- **PAX_KERNEXEC** — QAIC driver core in W^X kernel image; MHI channel-config tables (`aic100_channels`, `aic200_channels`), SAHARA opcode dispatch, and DRM ioctl-desc table in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `qaic_manage_ioctl`, `qaic_execute_bo_ioctl`, MHI event-handler entry, and SAHARA-packet handler.
- **PAX_REFCOUNT** — saturating `refcount_t` on `qaic_user`, `qaic_bo`, `qaic_drm_device` so concurrent partition-destroy + execute + close race cannot underflow into UAF.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `qaic_user`, `qaic_bo`, MANAGE scratch, per-DBC ring entries, and SAHARA memory-debug buffers; partition-key + per-partition VA-translation state zeroed on teardown.
- **PAX_UDEREF** — SMAP/PAN enforced on every QAIC ioctl; MANAGE transactions and per-slice descriptors copied via `copy_from_user` only.
- **PAX_RAP / kCFI** — `drm_driver.ioctls`, MHI client-ops vtables (SAHARA, SSR, RAS, TIMESYNC, IPCR), DBC submit/complete callbacks, and the GEM `prime_import` path marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of QAIC + MHI symbols behind CAP_SYSLOG; suppress `%p` of `qaic_device`, `qaic_user`, and DBC pointers in tracepoints + debugfs.
- **GRKERNSEC_DMESG** — restrict SAHARA-progress, SSR crash, RAS error, and MHI channel-state banners to CAP_SYSLOG so attackers cannot fingerprint device state via dmesg.
- **BO map CAP_SYS_ADMIN for cross-fd** — per-fd BO mmap allowed; cross-fd / cross-namespace BO sharing requires CAP_SYS_ADMIN + explicit dma-buf export.
- **MHI channel allowlist enforced** — device-side requests to open channels not in `aic100_channels[]`/`aic200_channels[]` refused at MHI core; defense against compromised-firmware sideband-channel attack.
- **SAHARA boot signature** — firmware blobs loaded via `request_firmware` are signed; SAHARA bootloader on-device verifies before exit; corrupted blobs fail boot, host enters fault state and refuses to register DRM device.
- **Partition-key MEMORY_SANITIZE** — partition-id, VA-translation cache, and per-partition semaphore state zeroed on `qaic_destroy_drm_device`; defeats key-reuse on partition re-create.
- **RAS event audit** — critical-severity RAS events emit `audit_log` records gated by CAP_AUDIT_READ; defeats silent fault corruption.
- **Debugfs RAS / DBC CAP_SYS_RAWIO** — register dumps + per-DBC state files gated CAP_SYS_RAWIO + 0400.

Rationale: QAIC is a PCIe bus-master whose firmware is loaded via an out-of-tree-signed sideband (SAHARA) and which speaks to the host over MHI — a protocol surface large enough that a compromised firmware can open sideband channels the driver never anticipated. Refcount-trapping `qaic_user`/`qaic_bo`, allowlisting MHI channels, MEMORY_SANITIZE on partition keys, mandatory SAHARA signature, and CAP_SYS_ADMIN on cross-fd BO sharing turn the AIC100 into a structurally constrained inference endpoint rather than a privileged bus-master proxy.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-partition NN-container internals (QC-proprietary)
- MHI core (separate Tier-3 in `drivers/bus/mhi/`)
- SAHARA bootloader on-device firmware
- AIC200-specific channel additions (post-baseline)
- Implementation code
