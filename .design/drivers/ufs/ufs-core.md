# Tier-3: drivers/ufs/core/{ufshcd,ufs-sysfs,ufs-mcq}.c — UFS Host Controller core (UTRD/UPIU + MCQ + sysfs)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/ufs/ufs-core.md
upstream-paths:
  - drivers/ufs/core/ufshcd.c
  - drivers/ufs/core/ufs-sysfs.c
  - drivers/ufs/core/ufs-mcq.c
  - drivers/ufs/core/ufshcd-priv.h
  - include/ufs/ufshcd.h
  - include/ufs/ufshci.h
  - include/ufs/ufs.h
-->

## Summary

Universal Flash Storage (UFS) Host Controller core — the shared driver that all UFSHCI-compliant host controllers (Qualcomm, Samsung, MediaTek, HiSilicon, Intel) plug into via `ufs_hba_variant_ops`. UFS itself is a JEDEC standard layered as: UFS Application Layer (UPIU — UFS Protocol Information Units carrying SCSI CDBs, NOP, query, RPMB, task-mgmt), UFS Transport Protocol (UTP — built around UTRD/UTMRD descriptor rings in host memory), and UniPro Link / M-PHY. ufshcd.c implements the SCSI low-level driver (`Scsi_Host_Template`), UPIU composition, UIC command engine for link-startup / power-mode / hibernate, error handler, runtime PM, and the dual-mode submission path (legacy single-doorbell ring vs. MCQ — Multi-Circular Queue introduced in UFSHCI 4.0).

This Tier-3 covers `ufshcd.c` (~11500 lines: SCSI integration, UPIU, UIC, EH, PM), `ufs-sysfs.c` (~2100 lines: per-attribute / per-flag / per-descriptor sysfs surface) and `ufs-mcq.c` (~730 lines: MCQ register programming, hwq alloc, IRQ + poll completion). Vendor variants (qcom, exynos, mediatek, hisi) and bsg/RPMB/crypto extensions are out of scope here.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ufs_hba` | per-host adapter state (regs, lrb pool, UTRD/UTMRD lists, MCQ hwqs, EH) | `drivers::ufs::Hba` |
| `struct ufshcd_lrb` | per-tag local reference block: cmd, ucd_req/rsp/prdt pointers, intr_cmd_status | `drivers::ufs::Lrb` |
| `struct utp_transfer_req_desc` | UTP transfer request descriptor (host-DMA visible) | `drivers::ufs::Utrd` |
| `struct utp_upiu_req` / `_rsp` | UPIU request / response (32B header + payload + EHS) | `drivers::ufs::Upiu` |
| `struct ufs_hw_queue` | per-MCQ hwq state: sq/cq DMA + head/tail doorbells | `drivers::ufs::HwQueue` |
| `ufshcd_init(hba, mmio, irq)` | per-controller probe entry; alloc UTRDL/UTMRDL, register SCSI host | `Hba::init` |
| `ufshcd_queuecommand(host, cmd)` | SCSI fast path: build LRB, compose UPIU, ring doorbell (or MCQ sq tail) | `Hba::queuecommand` |
| `__ufshcd_send_uic_cmd(hba, cmd)` / `ufshcd_send_uic_cmd(...)` | UIC engine (link-startup, power-mode, hibernate) | `Hba::send_uic_cmd` |
| `ufshcd_compose_devman_upiu(hba, cmd)` / `ufshcd_issue_devman_upiu_cmd(...)` | NOP / query / RPMB / task-mgmt UPIU | `Hba::issue_devman` |
| `ufshcd_query_flag(...)` / `_query_attr(...)` / `_read_desc(...)` | UFS device-manager query interface | `Hba::query_*` |
| `ufshcd_abort_one(req, priv)` / `ufshcd_abort_all(hba)` | per-tag abort + global abort path | `Hba::abort` |
| `ufshcd_eh_host_reset_handler(cmd)` | SCSI EH → UFS link recovery → host reset | `Hba::eh_host_reset` |
| `ufshcd_mcq_config_mac(hba, max)` / `ufshcd_mcq_req_to_hwq(...)` | MCQ MAC programming + per-request hwq routing | `Hba::mcq_*` |
| `ufshcd_mcq_init(hba)` / `ufshcd_mcq_enable(hba)` | MCQ bring-up: per-hwq SQ/CQ DMA + register programming | `Hba::mcq_init` |
| `ufshcd_compl_one_cqe(hba, hwq, cqe)` | MCQ completion path: parse CQE → release tag → scsi_done | `Hba::compl_one_cqe` |
| sysfs groups (`ufs_sysfs_default_groups`, `ufs_sysfs_lun_attributes_group`) | per-attribute / per-flag / per-descriptor sysfs surface | `Hba::sysfs_groups` |

## Compatibility contract

REQ-1: Per-controller MMIO region matches UFSHCI specification: capability + version + interrupt + host-control + UTRL/UTMRL base + door-bell + UIC command + MCQ extension registers. Each MMIO read/write goes through `ufshcd_readl/writel` (cross-ref `drivers/ufs/00-overview.md`).

REQ-2: Single-doorbell legacy mode: per-hba `nutrs`-deep UTRDL DMA ring; per-tag `ucd_req_ptr` / `ucd_rsp_ptr` / `ucd_prdt_ptr` populated; `REG_UTP_TRANSFER_REQ_DOOR_BELL` written with `1 << tag` to issue.

REQ-3: MCQ mode (UFSHCI 4.0+, `hba->mcq_sup && use_mcq_mode`): per-hwq SQ/CQ DMA rings + per-hwq SQ head/tail doorbells; `nr_hw_queues` programmed via blk-mq with poll/read/default partitions per `read_queues` / `poll_queues` / `rw_queues` module parameters.

REQ-4: UPIU types per JEDEC: NOP-OUT, NOP-IN, COMMAND, RESPONSE, DATA-OUT, DATA-IN, READY-TO-TRANSFER (RTT), TASK-MANAGEMENT-REQUEST, TASK-MANAGEMENT-RESPONSE, QUERY-REQUEST/RESPONSE, REJECT-UPIU. UTRD `command_type` selects DEV_CMD vs SCSI_CMD; `dd` flag selects no-data / write / read.

REQ-5: UIC commands per UniPro DME: DME_GET, DME_SET, DME_PEER_GET, DME_PEER_SET, DME_LINKSTARTUP, DME_HIBERNATE_ENTER/EXIT, DME_RESET, DME_ENABLE, DME_ENDPOINTRESET; per-command timeout default 500ms (`uic_cmd_timeout` module param, 500–5000ms inclusive).

REQ-6: Device-management query interface: per-IDN read/write of flags, attributes, descriptors (device / configuration / unit / interconnect / string / geometry / power / RPMB), with retry (default 3) + timeout (default 1500ms, configurable 1ms–30s via `dev_cmd_timeout`).

REQ-7: Error handler ladder: `ufshcd_check_errors` → fatal-only path schedules eh_work → `ufshcd_err_handler` runs link-recovery (UIC reset + hibernate exit + retry) → on failure, host-reset (HCE toggle + full re-init) → on persistent failure, SCSI EH escalation.

REQ-8: Runtime PM: per-hba `dev_pwr_mode` (ACTIVE / SLEEP / POWERDOWN) + `link_state` (ACTIVE / HIBERN8 / OFF); `ufshcd_runtime_suspend/_resume` walks gate-clk → hibernate → vreg-off; clock-gating + clock-scaling delegate to devfreq.

REQ-9: Crypto integration (`ufshcd-crypto.c`) — per-LRB inline-crypto keyslot programmed via `REG_CRYPTO_CFG`; gated on CAP.CRYPTO_SUPPORT.

REQ-10: Sysfs ABI under `/sys/bus/platform/drivers/ufshcd/.../`: `device_descriptor/`, `geometry_descriptor/`, `health_descriptor/`, `string_descriptors/`, `attributes/`, `flags/`, plus per-LUN groups. All read-only except a defined set of writable attrs (e.g. `wb_on`, `auto_hibern8_timer`).

REQ-11: RPMB unit (`ufs-rpmb.c`) bound via `bsg_setup_queue` providing a separate bsg device per RPMB W-LUN; per-RPMB transactions composed as ADVANCED-RPMB UPIU.

REQ-12: bsg general device (`ufs_bsg.c`) — per-host bsg device for raw UPIU passthrough (CAP_SYS_RAWIO required at open).

## Acceptance Criteria

- [ ] AC-1: `lsblk` shows UFS LUNs after probe on reference HW; SCSI sd device functional for fs mount.
- [ ] AC-2: `fio` randread/randwrite at >50k IOPS on MCQ-capable HW with `use_mcq_mode=1`.
- [ ] AC-3: `use_mcq_mode=0` cmdline regress to legacy single-doorbell path; same fs mount works.
- [ ] AC-4: Query API exercised via sysfs reads of `device_descriptor/manufacturer_name` and writable `attributes/wb_on` toggle.
- [ ] AC-5: bsg UPIU passthrough rejected without CAP_SYS_RAWIO; succeeds with CAP_SYS_RAWIO + valid UPIU.
- [ ] AC-6: Error injection (`ufs-fault-injection.c`) triggers EH → link recovery → command retry; no leaked tags after.
- [ ] AC-7: Runtime PM cycle: idle → autosuspend (2000ms default) → S0 resume; no data loss across N cycles.
- [ ] AC-8: Crypto on supported HW: dm-crypt over UFS LUN uses inline crypto; per-keyslot SCSI command issued with crypto config.

## Architecture

`Hba` lives in `drivers::ufs::Hba`:

```
struct Hba {
  mmio_base: NonNull<u8>,             // host-controller MMIO
  capabilities: u32,                  // REG_CONTROLLER_CAPABILITIES cache
  ufs_version: u32,
  mcq_capabilities: u32,
  mcq_sup: bool,
  nutrs: u16,                         // legacy UTRDL depth
  nr_hw_queues: u16,                  // MCQ hwq count (rw + read + poll)
  lrb_pool: KBox<[Lrb; MAX_NUTRS]>,
  utrdl: Dma<[Utrd]>,                 // legacy UTRDL ring
  utmrdl: Dma<[Utmrd]>,                // UTP task-management ring
  uhq: KBox<[HwQueue]>,               // per-MCQ-hwq state
  ufs_device_wlun: Option<Arc<ScsiDevice>>,
  uic_cmd_mutex: Mutex<()>,
  active_uic_cmd: Mutex<Option<&UicCommand>>,
  eh_work: WorkStruct,
  eh_wq: WorkqueueHandle,
  clk_list_head: Mutex<Vec<ClkInfo>>,
  vreg_info: VregInfo,
  dev_info: DeviceInfo,
  pwr_info: PowerInfo,                // gear, lane, pwr_mode
  ufshcd_state: Atomic<HbaState>,     // RESET / OPERATIONAL / ERROR / EH_SCHEDULED_FATAL
  link_state: Atomic<LinkState>,
  errors: u32,
  saved_err: u32,
  ufs_stats: UfsStats,
  vops: &'static UfsVariantOps,       // platform-specific hooks (qcom/exynos/...)
  crypto_capabilities: u32,
  crypto_cfg_register: u32,
}
```

Probe `Hba::init(mmio, irq)`:
1. `ufshcd_alloc_host` — alloc Scsi_Host + embedded `ufs_hba`.
2. `ufshcd_hba_enable` — toggle HCE; wait for `UFS_BIT(HCE)` set.
3. Read `REG_CONTROLLER_CAPABILITIES` → `nutrs`, `nutmrs`.
4. If `mcq_sup && use_mcq_mode`: `ufshcd_mcq_init` → alloc per-hwq SQ/CQ DMA rings; program per-hwq config registers; route blk-mq queue map.
5. `ufshcd_link_startup` → loop `ufshcd_dme_link_startup` UIC up to `DME_LINKSTARTUP_RETRIES` (3).
6. NOP-OUT loop until NOP-IN echoed (`NOP_OUT_RETRIES`).
7. `ufshcd_query_flag(fDeviceInit)` set; poll cleared with `FDEVICEINIT_COMPL_TIMEOUT` (1500ms).
8. Read device descriptor; populate `dev_info`.
9. `ufshcd_setup_clocks(true)`, `ufshcd_setup_vreg(true)`, init crypto if capable.
10. `scsi_add_host(host)`, `scsi_scan_host(host)` — LU discovery.
11. Init bsg (`ufs_bsg_probe`), RPMB (`ufs_bsg_probe_wlun`), sysfs groups.

Fast path `Hba::queuecommand(host, cmd)`:
1. `ufshcd_init_cmd_priv(host, cmd)` — populate `scsi_cmd_priv`.
2. Pick tag via blk-mq (or MCQ hctx). Locate `Lrb` from `lrb_pool[tag]`.
3. `ufshcd_init_lrb(hba, cmd)` — bind cmd → lrb, fill PRDT from sgl, set EHS length.
4. `ufshcd_comp_scsi_upiu(hba, lrb)` — build COMMAND UPIU (CDB inline, expected xfer len, flags).
5. Crypto: program LRB.crypto_key_slot if needed.
6. Legacy: `__set_bit(tag, &hba->outstanding_reqs); writel(1<<tag, REG_UTP_TRANSFER_REQ_DOOR_BELL)`.
7. MCQ: write CQE/SQE to `hwq->sqe_base[hwq->sq_tail_slot]`; bump SQ tail doorbell.

Completion (legacy): IRQ → `ufshcd_sl_intr` → `ufshcd_transfer_req_compl` walks completed-bits; per-tag `__ufshcd_transfer_req_compl` parses UPIU response, sets scsi_status, calls `scsi_done(cmd)`.

Completion (MCQ): IRQ → `ufshcd_mcq_intr` → per-hwq `ufshcd_mcq_poll_cqe_lock` drains CQE entries; `ufshcd_compl_one_cqe` decodes OCS + response → `scsi_done`.

UIC command engine `__ufshcd_send_uic_cmd(hba, cmd)`:
1. Acquire `uic_cmd_mutex`.
2. Write `REG_UIC_COMMAND_ARG_1/2/3`; write `REG_UIC_COMMAND` opcode.
3. Wait `IS_UIC_COMMAND_COMPL` interrupt OR poll completion under `uic_cmd_timeout`.
4. Read `REG_UIC_COMMAND_ARG_3` status (DME result).
5. Special-case link-startup, pwr-mode change, hibernate enter/exit per `ufshcd_uic_pwr_ctrl`.

Error handler `ufshcd_err_handler`:
1. `ufshcd_complete_requests` — drain in-flight completions.
2. `ufshcd_abort_all` — per-tag abort via task-mgmt UPIU; if any abort hangs, mark for host-reset.
3. If host-reset needed: HCE down + up, full re-init, restore PM state.
4. On persistent failure: SCSI EH escalation; on >`MAX_ERR_HANDLER_RETRIES`, give up.

Sysfs (`ufs-sysfs.c`): static groups for device / configuration / interconnect / geometry / health / power / string descriptors; per-attribute groups for `attributes/` and `flags/`; per-LUN groups under `host%d/target%d:0:%d/`.

## Hardening

(Inherits from `drivers/ufs/00-overview.md` § Hardening.)

ufs-core specific reinforcement:

- **UTRD/UTMRD DMA rings allocated coherent + non-swappable** — refuse to alloc above 32-bit DMA boundary if `dma_set_mask` rejects 64-bit; refuse if size > spec-max (32K entries).
- **Per-tag bitmap strictly serialized** — `outstanding_reqs` accessed under `host->host_lock`; double-issue of same tag triggers `WARN_ON` + EH escalation.
- **UIC command serialized via mutex** — concurrent DME ops impossible; `uic_cmd_timeout` upper-bounded at 5000ms.
- **Query API retry-bounded** — `QUERY_REQ_RETRIES` (3) hard limit; per-query timeout user-bounded 1ms–30s.
- **Crypto keyslot zeroed on free** — per-LRB keyslot config cleared after command completes; defense against key-leak across tags.
- **MCQ hwq depth derived from CAP.MQ_REGS_TYPE bits** — refuses to enable MCQ if HC capability undeclared.
- **Per-hwq SQ tail bumped under hwq spinlock** — concurrent submit across CPUs serialized per-hwq.
- **Error-handler rate-limited** — `MAX_ERR_HANDLER_RETRIES` (5) before giving up; per-EH cycle bounded in jiffies.
- **bsg passthrough requires CAP_SYS_RAWIO** + per-host bsg fd; refuses raw UPIU outside CAP scope.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist UTRD/UTMRD slabs and UPIU buffer caches; reject userland copies that straddle DMA region boundaries; bsg passthrough validated per-UPIU before copy_from_user.
- **PAX_KERNEXEC** — ufshcd entry points (`queuecommand`, IRQ handler, EH worker) live in `__ro_after_init` text; SCSI host template + `ufs_hba_variant_ops` marked const.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `ufshcd_queuecommand`, `ufshcd_send_uic_cmd`, `ufshcd_eh_host_reset_handler`, and MCQ CQE-completion entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `ufs_hba`, `scsi_device` (W-LUNs), bsg cdev refs; overflow trap defeats EH/PM race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for UPIU req/rsp pools, PRDT arrays, query buffers, crypto keyslot memory, and UTRDL/UTMRDL allocations.
- **PAX_UDEREF** — SMAP/PAN enforced on every bsg / RPMB / sysfs entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `ufs_hba_variant_ops` (`init`, `setup_clocks`, `hce_enable_notify`, `link_startup_notify`, `pwr_change_notify`, `dbg_register_dump`) marked `__ro_after_init`; SCSI host template indirect dispatch typed.
- **GRKERNSEC_HIDESYM** — suppress `%p` of `ufs_hba`, UTRD pointers, MCQ register bases in tracepoints; gate kallsyms behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict UFS link-startup failure, UIC error, OCS-error banners and PRDT-dump output to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict** — writable sysfs attrs (`wb_on`, `auto_hibern8_timer`, `enable_clk_gating`, fault-injection knobs) require CAP_SYS_ADMIN.
- **CAP_SYS_RAWIO on bsg** — raw UPIU passthrough on `/dev/bsg/ufs-bsg*` and RPMB W-LUN bsg devices require CAP_SYS_RAWIO; refuse open without it.
- **Security-protocol RPMB allowlist** — SECURITY-PROTOCOL-IN/OUT UPIUs accepted only with protocol-id in RPMB allowlist (0xEC for RPMB); other protocol-ids rejected.
- **HCI MMIO range checked** — every `ufshcd_readl/writel` validated against the platform-declared MMIO region; reject offsets above declared length to defeat platform-quirk MMIO walks.
- **sg-list length bounded** — per-LRB PRDT length capped at HC-reported max; refuse `scsi_for_each_sg` walk that would exceed PRDT slot count.
- **Query CAP_SYS_ADMIN gate** — non-read query writes through sysfs/bsg gated by CAP_SYS_ADMIN in the owning user namespace.

Rationale: UFS HCI exposes a SCSI device + an inline-crypto engine + a RPMB security boundary + a raw UPIU passthrough channel — every one of those is a direct path to silent data exfiltration or rootkit-grade persistence (RPMB-stored bootloader signing). RAP/kCFI on the variant-ops vtable, CAP_SYS_RAWIO on bsg, RPMB protocol-allowlisting, memory-sanitize on UPIU and key buffers, and refcount-overflow trapping turn UFS from "trust whatever JEDEC firmware says" into a structural enforcement boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Vendor host drivers (qcom-ufs, exynos-ufs, mediatek-ufs, hisi-ufs — separate Tier-3s)
- HPB (Host Performance Booster) extension (future)
- Crypto details (covered in `ufs-crypto.md` future Tier-3)
- RPMB UPIU details (covered in `ufs-rpmb.md` future Tier-3)
- bsg passthrough framework (covered in `block/bsg.md`)
- Implementation code
