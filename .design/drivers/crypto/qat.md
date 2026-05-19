# Tier-3: drivers/crypto/intel/qat/ — Intel QuickAssist Technology (multi-engine crypto + compression offload, SR-IOV VFs)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/crypto/00-overview.md
upstream-paths:
  - drivers/crypto/intel/qat/qat_common/adf_accel_devices.h
  - drivers/crypto/intel/qat/qat_common/adf_accel_engine.c
  - drivers/crypto/intel/qat/qat_common/adf_admin.c
  - drivers/crypto/intel/qat/qat_common/adf_aer.c
  - drivers/crypto/intel/qat/qat_common/adf_cfg.c
  - drivers/crypto/intel/qat/qat_common/adf_ctl_drv.c
  - drivers/crypto/intel/qat/qat_common/adf_dev_mgr.c
  - drivers/crypto/intel/qat/qat_common/adf_sriov.c
  - drivers/crypto/intel/qat/qat_common/adf_transport.c
  - drivers/crypto/intel/qat/qat_common/adf_vf_isr.c
  - drivers/crypto/intel/qat/qat_common/qat_algs.c
  - drivers/crypto/intel/qat/qat_common/qat_asym_algs.c
  - drivers/crypto/intel/qat/qat_common/qat_crypto.c
  - drivers/crypto/intel/qat/qat_common/qat_compression.c
  - drivers/crypto/intel/qat/qat_common/qat_comp_algs.c
  - drivers/crypto/intel/qat/qat_common/qat_hal.c
  - drivers/crypto/intel/qat/qat_common/qat_uclo.c
  - drivers/crypto/intel/qat/qat_common/qat_mig_dev.c
  - drivers/crypto/intel/qat/qat_common/adf_rl.c
  - drivers/crypto/intel/qat/qat_common/adf_telemetry.c
  - drivers/crypto/intel/qat/qat_4xxx/adf_drv.c
  - drivers/crypto/intel/qat/qat_420xx/adf_drv.c
  - drivers/crypto/intel/qat/qat_6xxx/
  - drivers/crypto/intel/qat/qat_c3xxx/adf_drv.c
  - drivers/crypto/intel/qat/qat_c62x/adf_drv.c
  - drivers/crypto/intel/qat/qat_dh895xcc/adf_drv.c
  - drivers/crypto/intel/qat/qat_c3xxxvf/adf_drv.c
  - drivers/crypto/intel/qat/qat_c62xvf/adf_drv.c
  - drivers/crypto/intel/qat/qat_dh895xccvf/adf_drv.c
  - include/linux/qat/qat_mig_dev.h
-->

## Summary

Intel QuickAssist Technology (QAT) is an in-package multi-engine accelerator providing symmetric/asymmetric crypto AND lossless compression offload. The driver tree, often called `adf` ("Acceleration Driver Framework") in upstream symbol naming, supports five+ silicon generations:

- **DH895xcc** — first-gen PCIe accelerator, c2xxx series
- **C3xxx / C62x** — Atom-class SoC integrated accelerators, with SR-IOV VFs (`qat_c3xxxvf`, `qat_c62xvf`)
- **4xxx (Gen4 / "QAT Gen4")** — Sapphire Rapids generation, on-die in Xeon CPUs, SR-IOV VFs, RAS, telemetry, rate-limiting
- **420xx (Gen4.2)** — Emerald Rapids / refinements
- **6xxx (Gen6)** — next-gen, in development

The framework structure: every silicon family ships a small `qat_<silicon>/adf_drv.c` (PCI probe + per-silicon `hw_data` filling out vtables) and links against `qat_common/` which provides the bulk of the driver (algorithm registration, transport rings, firmware loader, admin mailbox, RAS, telemetry, rate-limiting, SR-IOV, migration).

QAT registers with the kernel crypto API for: AES-{CBC,CTR,XTS,GCM,CCM}, ChaCha20-Poly1305, SHA-{1,224,256,384,512}, SHA3, RSA, DH, ECDH, ECDSA, plus zlib/deflate, LZ4, and zstd compression. Each algorithm carries priority sufficient to win over software fallback.

Gen4+ silicon also exposes live-migration support for VFs (`qat_mig_dev`) — a VF's state can be saved/restored to enable live-migration of a guest using a QAT VF.

This Tier-3 covers `qat_common/` framework + per-family glue with focus on the configuration-file mechanism (`qat_service`), the transport rings, firmware loader (UCLO), SR-IOV / PFVF mailbox, and VFIO-mediated migration.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct adf_accel_dev` | per-QAT-device control block (one per PF or VF) | `drivers::qat::AccelDev` |
| `struct adf_hw_device_data` | per-silicon vtable (4xxx vs 420xx vs c62x ...) | `drivers::qat::HwDeviceData` |
| `struct adf_accel_pci` | per-PCI bus context (PF-only) | `drivers::qat::AccelPci` |
| `struct adf_etr_data` | per-device ETR (Eagletail Ring) transport state | `drivers::qat::EtrData` |
| `struct adf_etr_ring_data` | per-bank-ring HW ring | `drivers::qat::EtrRing` |
| `adf_dev_init(accel_dev)` / `adf_dev_start(accel_dev)` / `adf_dev_stop(accel_dev)` / `adf_dev_shutdown(accel_dev)` | per-device lifecycle | `AccelDev::init` / `_start` / `_stop` / `_shutdown` |
| `adf_devmgr_add_dev(accel_dev, pf)` / `_rm_dev(...)` | per-device manager registration | `DevMgr::add_dev` / `_rm_dev` |
| `adf_send_admin(accel_dev, req, resp, ae_mask)` | per-AE admin mailbox submit | `AccelDev::send_admin` |
| `adf_cfg_dev_add(accel_dev)` / `_add_key_value_param(...)` | per-device configuration | `AccelDev::cfg_add` / `_add_key_value` |
| `adf_ctl_ioctl(fp, cmd, arg)` | `/dev/qat_adf_ctl` ioctl entry | `Subsystem::ctl_ioctl` |
| `adf_sriov_configure(pdev, numvfs)` | enable/disable SR-IOV VFs | `AccelPci::sriov_configure` |
| `adf_iov_send_resp(work)` | PF→VF PFVF mailbox response worker | `AccelDev::iov_send_resp` |
| `qat_alg_skcipher_register` / `qat_alg_aead_register` / `qat_asym_algs_register` / `qat_comp_algs_register` | per-class registration | `AccelDev::register_*` |
| `qat_uclo_map_obj(...)` / `qat_uclo_wr_all_uimage(...)` | firmware loader (UCLO blob parsing + AE upload) | `AccelDev::uclo_load` |
| `qat_hal_set_ae_ctx_mode` / `_init_gpr` / `_clr_csr_ae` / ... | per-AE register programming via HAL | `AccelDev::hal_*` |
| `adf_transport_init(accel_dev)` / `_create_ring(...)` / `_send_msg(...)` | per-ring transport | `EtrData::init` / `_create_ring` / `_send_msg` |
| `adf_init_aer` / `adf_handle_pci_aer` | AER (PCIe error recovery) handler | `AccelPci::aer_handle` |
| `adf_rl_init(accel_dev)` / `adf_rl_send_admin_init` | rate-limiting (Gen4+) | `AccelDev::rl_init` |
| `adf_telemetry_init(accel_dev)` / `adf_telemetry_start(...)` | per-device telemetry (Gen4+) | `AccelDev::telemetry_init` |
| `adf_vf_isr_resource_alloc` / `adf_vf2pf_*` | per-VF PFVF mailbox client | `AccelDev::vf_pfvf_*` |
| `qat_vfmig_create(pdev, vf_id)` / `_save_state` / `_load_state` | VF live-migration state machine | `MigDev::create` / `_save_state` / `_load_state` |
| `adf_anti_rb_init(accel_dev)` | anti-replay-buffer (RAS) | `AccelDev::anti_rb_init` |
| `adf_dbgfs_init(accel_dev)` | debugfs root | `AccelDev::dbgfs_init` |

## Compatibility contract

REQ-1: Per-silicon-family `adf_drv.c` registers PCI IDs (4xxx: `0x4940`, c62x: `0x37c8`, dh895xcc: `0x435`, …) and instantiates a `struct adf_hw_device_data` filling per-family bank counts, AE counts, ring offsets, FW image names, and CSR layouts.

REQ-2: Per-device probe sequence: BAR ioremap → `adf_devmgr_add_dev` → `adf_dev_init` (allocates rings, parses cfg) → `adf_dev_start` (uploads FW, enables AEs, registers algorithms) → ready.

REQ-3: Configuration: per-device `/etc/<device>_dev<id>.conf` parsed by userspace `qatlib` and pushed in via `/dev/qat_adf_ctl` ioctl `IOCTL_DEV_CONFIG` — adds key-value pairs to `accel_dev->cfg`. Cfg keys include `ServicesEnabled` (`sym;asym;dc;sym;asym;dc`), per-bank service assignment, ring count, polling vs interrupt mode.

REQ-4: Per-AE firmware upload via UCLO (Ucode Loader) format: `qat_<silicon>.bin` + `qat_<silicon>_mmp.bin` blobs loaded via `request_firmware`; UCLO parser maps AE microcode regions; `qat_uclo_wr_all_uimage` writes per-AE microstore RAM via HAL.

REQ-5: Transport: ETR (Eagletail Ring) banks per AE, each bank has request + response rings; producer writes request descriptor + writes tail register; FW consumes; FW writes response descriptor + writes response tail; consumer reads.

REQ-6: Algorithm registration: `qat_alg_register` registers all skcipher + aead variants per-device; second device of same type does NOT double-register (`active_devs` ref-counted).

REQ-7: SR-IOV: `adf_sriov_configure(pdev, numvfs)` enables N VFs; per-VF assigned a subset of bank-rings; PF→VF mailbox via PFVF protocol; VF driver lives in `qat_<silicon>vf/`.

REQ-8: PFVF mailbox (Gen2 + Gen4): per-VF MMIO mailbox register; PF sees VF→PF interrupts and dispatches via `adf_iov_send_resp` workqueue; protocol negotiates version + ring assignment + ras notifications.

REQ-9: AER (PCIe Advanced Error Reporting) integration: per-device `pci_error_handlers`; on detected error: stop algorithms, drain rings, reset device via FLR, re-init, re-register algorithms.

REQ-10: Rate Limiting (Gen4+): per-class throughput cap configured via sysfs + admin mailbox; FW enforces; debugfs reports observed throughput.

REQ-11: Telemetry (Gen4+): per-device telemetry buffer DMA-mapped; FW writes per-tick counters; userspace reads via debugfs `/sys/kernel/debug/qat_4xxx_<bdf>/telemetry/`.

REQ-12: VF live-migration (Gen4+): `qat_vfmig_create(pdev, vf_id)` constructs `qat_mig_dev`; `vfio_pci_core` calls `_save_state` to snapshot VF state (rings, FW state, ETR queues) into an opaque blob; destination calls `_load_state` to restore. Used by vfio + iommufd live-migration of a guest with QAT VF passthrough.

REQ-13: `/dev/qat_adf_ctl` mode 0600 (root-only); all ctl ioctls (`IOCTL_DEV_CONFIG`, `IOCTL_DEV_START`, `IOCTL_DEV_STOP`, `IOCTL_GET_NUM_DEVS`, `IOCTL_STATUS_ACCEL_DEV`) require CAP_SYS_ADMIN.

## Acceptance Criteria

- [ ] AC-1: `lspci -nn -d 8086:` lists QAT PF; `dmesg | grep -i qat` shows probe, UCLO firmware load, AE start, algorithm registration.
- [ ] AC-2: `qatconfig --device 0 --conf <path>` succeeds; per-device cfg applied; `/sys/kernel/debug/qat_4xxx_<bdf>/state` shows `up`.
- [ ] AC-3: `cat /proc/crypto | grep qat` lists registered skcipher + aead + asym + compression algorithms with priority > software fallback.
- [ ] AC-4: `cryptsetup benchmark` shows QAT-AES-XTS throughput improvement vs AES-NI on Sapphire Rapids.
- [ ] AC-5: SR-IOV enable: `echo 16 > /sys/class/pci_bus/.../numvfs` creates 16 VFs visible to lspci; PFVF mailbox handshake completes; per-VF rings assigned.
- [ ] AC-6: vfio-pci passthrough of QAT VF to KVM guest: guest sees device; in-guest `qat_4xxxvf` driver probes; in-guest crypto API picks QAT VF algorithms.
- [ ] AC-7: VF live-migration: source guest with QAT VF migrated to destination host; in-flight crypto ops complete or fail-and-retry cleanly with no data loss.
- [ ] AC-8: AER recovery: `setpci` forced PCI error → AER handler stops, resets via FLR, restarts; algorithms re-registered with same name (no userland disruption).
- [ ] AC-9: Telemetry test: `/sys/kernel/debug/qat_4xxx_<bdf>/telemetry/control` set to 1; counters non-zero after stress; Rate-Limiting test: per-service throughput cap honored.

## Architecture

`AccelDev` is the per-device container:

```
struct AccelDev {
  accel_id: u32,
  hw_device: KBox<HwDeviceData>,
  accel_pci_dev: AccelPci,
  fw_loader: Option<KBox<UcloHandle>>,
  cfg: Mutex<Cfg>,
  transport: KBox<EtrData>,
  admin: KBox<AdminComms>,
  ras: RasState,
  rl: Option<KBox<RateLimit>>,
  telemetry: Option<KBox<Telemetry>>,
  vf_info: Option<Vec<VfInfo>>,
  ae_num: u32,
  ring_per_bank: u32,
  state: AccelState,        // DOWN / INITIALIZED / STARTED / SHUTDOWN_FAILED
  refcount: Refcount,
}

struct HwDeviceData {
  dev_class: AccelDevClass,        // 4xxx / 420xx / c62x / c3xxx / dh895xcc
  num_banks: u32,
  num_rings_per_bank: u32,
  num_aes: u32,
  fw_name: &'static str,
  fw_mmp_name: &'static str,
  default_cfg: KBox<ServiceConfig>,
  ops_init_device: fn(&mut AccelDev) -> Result<(), Err>,
  ops_send_admin_init: fn(&mut AccelDev) -> Result<(), Err>,
  ops_get_ae_mask: fn(&AccelDev) -> u64,
  ops_get_arb_mapping: fn(&AccelDev) -> &[u32],
  ops_get_etr_bar_id: fn() -> u32,
  ops_get_pf2vf_offset: Option<fn(u32) -> u32>,
  // ... ~50 more per-silicon function pointers
}
```

PF probe sequence (`adf_4xxx_probe`):
1. `pci_enable_device(pdev)` + DMA mask negotiate (64-bit).
2. Alloc `AccelDev` + per-family `HwDeviceData` (`adf_init_hw_data_4xxx`).
3. `adf_devmgr_add_dev(accel_dev, NULL)` — register in PF table.
4. `pci_request_regions` + ioremap BAR0 (CSR) + BAR2 (ETR rings).
5. `adf_dev_init(accel_dev)`:
   - `adf_admin_init` — allocate admin mailbox DMA buffers.
   - `adf_transport_init` — alloc per-bank rings.
   - `qat_crypto_dev_config` — populate default cfg if no userland cfg.
6. Wait for `IOCTL_DEV_START` from userland (or auto-start on Gen4) → `adf_dev_start`:
   - `qat_uclo_map_obj` — parse UCLO blob.
   - `qat_uclo_wr_all_uimage` — upload microcode to each AE.
   - `adf_send_admin_init` — boot each AE, wait completion.
   - `qat_alg_register` + `qat_asym_algs_register` + `qat_comp_algs_register` — register algorithms.
   - `state = STARTED`.

Request flow (`qat_alg_skcipher_encrypt`):
1. Per-`crypto_request` → build per-cmd descriptor in DMA-coherent slab.
2. DMA-map scatterlist (`qat_bl_*`).
3. Select target bank-ring (per-CPU affinity hint).
4. Build per-ring fw_req: opcode + service-id + content-descriptor (key/iv schedule) + SGL pointers.
5. Append to ring + write tail CSR.
6. Return `-EINPROGRESS`.
7. IRQ (or poll mode) → response ring head advances → `qat_alg_callback` → dma-unmap → `crypto_request_complete`.

PFVF mailbox flow (PF side):
1. VF triggers interrupt via PF MISC IRQ.
2. PF reads PFVF source register, identifies VF id.
3. Queue `adf_iov_send_resp` work on `pf2vf_resp_wq`.
4. Worker calls per-silicon handler → reads VF message → composes response → writes PF→VF MMIO register → raises VF IRQ.
5. VF wakes on IRQ → reads response.

UCLO firmware load (`qat_uclo_map_obj`):
1. Validate UCLO header (magic, version, AE count) against silicon family.
2. For each per-AE microcode region: validate target AE id, target memory region (`uCstore`, `Sram`, `Dram`), bounds.
3. For each region: HAL writes to AE microstore via `qat_hal_wr_uwords_to_page` (paginated bursts).
4. Apply per-AE patches (e.g., for known errata).
5. Set AE context start mask → AE boots.

VF migration save (`qat_vfmig_save_state`):
1. Pause VF — block new requests via PFVF.
2. Drain in-flight on VF bank rings (wait for ring head==tail).
3. Serialize VF state: ring head/tail/base, AE GPR state, ETR config, RL config.
4. Encode to a versioned migration blob.
5. Return blob to vfio-pci core for transmission.

VF migration load (`qat_vfmig_load_state`):
1. Decode blob, validate version + checksums.
2. Reset target VF.
3. Restore ring base/head/tail, AE context, ETR cfg, RL cfg.
4. Resume VF.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `etr_ring_no_oob` | OOB | per-ring producer/consumer indices wrap correctly within ring size; no overflow. |
| `admin_buf_no_oob` | OOB | admin request/response buffers DMA-sized and bounded by `ADF_ADMINMSG_LEN`. |
| `uclo_load_no_oob` | OOB | UCLO region writes bounded by per-AE microstore size; refuse out-of-bounds region. |
| `pfvf_no_uaf` | UAF | VF-info entries lifetime bound by `accel_dev` refcount; PFVF worker observes valid VF. |
| `cfg_kv_no_oob` | OOB | per-cfg `adf_user_cfg_key_val[].key`/`.val` strictly bounded by `ADF_CFG_MAX_KEY_LEN_IN_BYTES`. |

### Layer 2: TLA+

`models/qat/pfvf_mailbox.tla` (parent-declared): proves PFVF mailbox handshake (VF→PF interrupt + PF→VF response + per-message version negotiation) progresses with no deadlock; per-VF restart-on-FLR re-syncs state.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post-`adf_dev_start`: `state == STARTED` AND every registered algorithm visible via `crypto_find_alg` | `AccelDev::start` |
| Per-`qat_alg_skcipher_encrypt`: per-request descriptor freed exactly once on completion (success or error) | `AccelDev::encrypt` |
| Per-`qat_uclo_wr_all_uimage`: every per-AE microstore region written exactly once; CRC verified | `AccelDev::uclo_load` |
| Per-`qat_vfmig_save_state` → `_load_state` round-trip: every byte of VF state recovered; ring base/head/tail consistent | `MigDev::save_state` / `_load_state` |

### Layer 4: Verus/Creusot functional

End-to-end: userland AF_ALG client requesting `xts(aes)` → kernel resolves to qat-aes-xts (priority 4001) → request dispatched to bank-ring → AE microcode executes XTS → response ring populated → IRQ → completion → userland gets correct ciphertext (bit-exact match against tcrypt KAT vectors).

## Hardening

(Inherits row-1 features from `drivers/crypto/00-overview.md` § Hardening.)

QAT-specific reinforcement:

- **UCLO firmware signature** — `request_firmware` for `qat_<silicon>.bin` / `_mmp.bin` enforces `CONFIG_FW_LOADER_SIG_FORCE=y`; UCLO header version + per-region checksums validated before AE upload.
- **`/dev/qat_adf_ctl` mode 0600 + CAP_SYS_ADMIN** — only root-with-CAP_SYS_ADMIN may push cfg or start/stop devices.
- **Per-cfg key-value bounds** — `adf_add_key_value_data` clamps key + value lengths; refuse oversize.
- **Per-AE microstore write bounds** — `qat_hal_wr_uwords_to_page` refuses writes outside microstore.
- **PFVF protocol version negotiated** — refuse VF messages with mismatched version; legacy VFs upgraded via PF response.
- **Per-VF bank-ring assignment isolated** — PF enforces non-overlapping ring assignment per VF; refuse double-assignment.
- **VF live-migration blob versioned + checksummed** — destination refuses mismatched blob; corrupted blob aborts migration.
- **AER recovery serialized via global lock** — concurrent AER + cfg + start serialized; no half-recovered state.
- **Telemetry buffer DMA-mapped read-only from FW side** — no FW write-back to host structures beyond telemetry region.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `adf_accel_dev`, `adf_etr_data`, `adf_etr_ring_data`, admin-mailbox DMA buffers, UCLO staging, and per-cfg key-value slabs; `/dev/qat_adf_ctl` ioctl `copy_from_user` clamped to `sizeof(struct adf_user_cfg_*)`; scatterlist `qat_bl_*` clamps source/dest sizes.
- **PAX_KERNEXEC** — per-silicon `hw_device_data` vtables, `adf_ctl_ops` file_operations, AER handler chain, PFVF handler dispatch, and UCLO loader entry points live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `adf_ctl_ioctl`, `qat_alg_skcipher_encrypt`, `adf_send_admin`, UCLO loader, and PFVF worker entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `accel_dev`, per-VF info, per-ring `etr_ring_data`, per-algorithm-active-devs counters, and `qat_mig_dev` lifetime; overflow trap defeats SR-IOV teardown races and VF migration UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-cmd descriptors, admin mailbox buffers, content-descriptor slabs (which contain expanded key schedule), UCLO staging, and VF-migration state blobs; key material never bleeds across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every `/dev/qat_adf_ctl` ioctl, sysfs write, debugfs entry, and AF_ALG dispatch reaching QAT algorithms.
- **PAX_RAP / kCFI** — `hw_device_data` per-silicon vtable indirect calls (50+ function pointers per device), `adf_ctl_ops`, `pci_error_handlers`, PFVF protocol dispatchers, and UCLO callback tables kCFI-typed.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of admin mailbox MMIO base, per-AE microstore pointer, UCLO buffer addresses, and ETR ring base behind CAP_SYSLOG; tracepoint `%p` suppressed.
- **GRKERNSEC_DMESG** — restrict QAT-error, RAS-violation, PFVF-failure, AER-recovery, UCLO-load-fail banners to CAP_SYSLOG; defeat probing AE state via dmesg.
- **CAP_SYS_ADMIN strict on cfg-file ingestion** — `/dev/qat_adf_ctl` `IOCTL_DEV_CONFIG` requires CAP_SYS_ADMIN in init userns; cfg key allowlist enforced (refuse unknown keys); per-cfg-value length clamped.
- **VF passthrough via vfio-pci + iommufd** — QAT VF passthrough refused unless iommufd domain attaches the VF; raw VFIO group-mode disabled by default per `drivers/vfio/00-overview.md`.
- **UCLO firmware signature mandatory** — refuse `qat_<silicon>.bin` / `qat_<silicon>_mmp.bin` not signed by Intel vendor key; per-region CRC validated.
- **Scatterlist clamp** — `qat_bl_*` strictly clamps source/dest sizes against per-algorithm max-input; refuse oversize SGLs that would overflow descriptor pointer arrays.
- **VF migration blob signed + version-locked** — destination refuses mismatched-version or unsigned blobs; defense against migration-state tampering across hosts.
- **Telemetry / RL knobs CAP_SYS_ADMIN** — debugfs telemetry control + RL sysfs writers require CAP_SYS_ADMIN in init userns.
- **Per-VF ring assignment audited** — PF refuses overlapping ring assignment that would let one VF read/write another VF's response ring.

Rationale: QAT VFs are the most-deployed crypto offload in modern data centers (TLS termination, IPsec, ZFS-on-Linux compression), and a single UCLO firmware corruption, PFVF protocol confusion, or VF-migration-blob tampering compromises every guest using the accelerator. RAP/kCFI on the 50+ function-pointer per-silicon vtable, firmware signature enforcement, CAP_SYS_ADMIN on cfg/ctl, iommufd-required passthrough, refcount-overflow trapping, and explicit key-schedule zeroization turn QAT from "trusted because Intel signs the FW" into a structural multi-tenant boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-silicon UCLO blob format (Intel proprietary; covered by Intel UCLO spec)
- Userspace `qatlib` library (covered out-of-tree at github.com/intel/qatlib)
- VFIO core (covered in `drivers/vfio/00-overview.md` future Tier-3)
- IOMMUFD device passthrough (covered in `.design/drivers/iommu/iommufd-device.md`)
- Compression-only flows for non-crypto consumers (covered in `crypto/acompress.md` future)
- 32-bit-only paths (QAT exists on x86-64 only)
- Implementation code
