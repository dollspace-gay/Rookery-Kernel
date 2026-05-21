# Tier-3: drivers/scsi/megaraid/{megaraid_sas_base,megaraid_sas_fusion,megaraid_sas_fp,megaraid_sas_debugfs}.c — LSI / Broadcom MegaRAID SAS (every Dell PERC + HP P-Series RAID + Lenovo TS RAID + supermicro)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/scsi/00-overview.md
sibling: drivers/scsi/aacraid.md
sibling: drivers/scsi/mpt3sas.md
upstream-paths:
  - drivers/scsi/megaraid/megaraid_sas_base.c                 (~9145 lines: SCSI host template, init, MFI cmd path, ioctl, sysfs)
  - drivers/scsi/megaraid/megaraid_sas_fusion.c               (~5380 lines: Fusion silicon — IT/IR-Fusion DMA-direct path)
  - drivers/scsi/megaraid/megaraid_sas_fp.c                   (~1425 lines: Fast-Path — driver-side LD I/O bypass of FW for RAID-0/1/5/6)
  - drivers/scsi/megaraid/megaraid_sas_debugfs.c              (debugfs surface)
  - drivers/scsi/megaraid/megaraid_sas.h
  - drivers/scsi/megaraid/megaraid_sas_fusion.h
  - drivers/scsi/megaraid/mbox_defs.h
-->

## Summary

megaraid_sas is the driver for LSI/Broadcom MegaRAID SAS HBAs — the dominant enterprise RAID HBA family worldwide. Every Dell PERC (PowerEdge RAID Controller) shipping in PowerEdge servers since the mid-2000s, every HP P-series RAID controller in ProLiant, every Lenovo TS RAID, Supermicro Aero RAID, IBM ServeRAID-M, and countless OEM-rebranded MegaRAID variants all use this driver. Covers MegaRAID SAS-1/2/3/4 silicon: SAS1078, SAS2008/2108, SAS2208/2308, SAS3008/3108, SAS3216/3224, SAS3408/3416/3416E, SAS3508/3516, SAS3708/3716 (Aero), and modern SAS3908 (Crusader). Also covers Tri-mode SAS+SATA+NVMe controllers.

Architecturally distinct from `mpt3sas` (its sibling SAS HBA driver):
- **mpt3sas** = SAS HBA in IT (Initiator Target) mode — bare-disk passthrough, no RAID FW.
- **megaraid_sas** = RAID HBA in IR (Integrated RAID) mode — RAID FW managing logical volumes.

Same silicon can be flashed for either firmware mode; the LSI/Broadcom IT-firmware = mpt3sas, IR-firmware = megaraid_sas. Many enterprise deployments use IR-mode for RAID-protected boot drives + IT-mode HBAs for general data.

Two distinct silicon families:
- **Legacy MFI (MegaRAID Firmware Interface)** silicon — pre-SAS-3 (1078/2008/2108/2208/2308). MFI command set, mailbox-style transport. ~9145 lines in `megaraid_sas_base.c`.
- **Fusion (DMA-direct)** silicon — SAS-3 onwards (3008+). Fusion DMA — driver writes IO descriptors directly via DMA-mapped MMIO; no FW mailbox round-trip for fast-path IO. ~5380 lines in `megaraid_sas_fusion.c`.

Plus Fast-Path (`megaraid_sas_fp.c`, 1425 lines): for RAID-0/1/5/6 LDs (Logical Drives), driver can bypass FW entirely on read/write — driver computes physical-disk addresses + issues raw SCSI IO directly. Provides ~30% IO latency reduction vs going through FW.

This Tier-3 covers ~16100 lines.

## Rust translation posture

megaraid_sas has the same pattern as the other RAID HBAs (typed FIB/MFI cmd, /dev/megaraid_sas ioctl, /dev/megaraid_sas chardev) but adds the Fusion DMA-direct fast-path complexity. Translation:

- **`megaraid-sas-core` crate** — PCI lifecycle, SCSI host template, MFI cmd transport, ioctl/sysfs surface.
- **Typed MFI cmd interface.** Each opcode a typed struct.
- **`megaraid-sas-fusion` sub-crate** — Fusion DMA-direct path. Per-CPU reply queue, MPI-2-derived IO descriptors.
- **`megaraid-sas-fp` sub-crate** — Fast-Path LD lookup + SCSI command translation to direct-disk SCSI; bypass of FW for RAID-0/1/5/6.
- **MegaRaidAdapter phase typing.** `MegaRaidAdapter<Probed → MfiReady → FusionReady → Online>` (silicon-dependent path).
- **/dev/megaraid_sas chardev** ioctls — destination of `storcli`, `MegaCli`, `perccli` vendor tools. Major historical CVE surface.

Grsec is mandatory. megaraid_sas has had multiple CVEs: CVE-2021-39685 missing CAP check on megasas_mgmt_ioctl, CVE-2019-19528 stack OOB read in MFI cmd parsing, CVE-2020-12114 unsafe pointer deref in legacy ioctl path.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct megasas_instance` | per-HBA state | `MegaRaidAdapter<Phase>` |
| `struct megasas_cmd` | MFI cmd | `MfiCmd<Op>` |
| `struct megasas_cmd_fusion` | Fusion IO descriptor | `FusionIo` |
| `struct megasas_register_set` | HW register set | `RegSet` |
| `megasas_probe_one(pdev, ent)` / `_detach_one(pdev)` | PCI probe/remove | `MegaRaidAdapter::probe` / `Drop` |
| `megasas_init_adapter_*` (mfi vs fusion) | per-silicon init | `MegaRaidAdapter::init_*` |
| `megasas_issue_blocked_cmd(...)` / `_issue_polled(...)` | MFI sync cmd | `MfiCmd::issue_sync` / `_polled` |
| `megasas_build_and_issue_cmd(scsi_cmd)` / `_build_io_fusion(...)` / `_build_dcdb_fusion(...)` | scsi cmd issue (MFI vs Fusion) | `Scsi::build_and_issue` |
| `megasas_complete_cmd(...)` / `_complete_cmd_dpc(...)` | completion | `Completion::*` |
| `megasas_isr(...)` / `_isr_fusion(...)` | IRQ handlers | `Irq::handle_*` |
| `megasas_fusion_update_can_queue(...)` | per-LD queue depth | `Fusion::update_canq` |
| `megasas_get_ld_io_request(...)` / `_get_request_descriptor(...)` | descriptor build | `Fusion::build_descriptor` |
| `megasas_build_ld_nonrw_fusion(...)` / `_build_ldio_fusion(...)` / `_build_syspd_fusion(...)` | Fusion IO build per IO type | `Fusion::build_*` |
| `megasas_set_pd_lba(...)` / `_io_attempt_fast_path(...)` | Fast-Path translation | `FastPath::translate` |
| `megasas_check_raid_map(...)` | RAID map refresh | `RaidMap::check` |
| `megasas_mgmt_ioctl(...)` / `_mgmt_compat_ioctl(...)` | /dev/megaraid_sas chardev | `Mgmt::ioctl` |
| `megasas_mgmt_fw_ioctl(...)` | MFI passthru | `Mgmt::fw_passthru` |
| `megasas_aen_polling(work)` | AEN (async event) poller | `Aen::poll` |
| `megasas_register_aen(...)` | AEN subscriber | `Aen::register` |
| `megasas_reset_targets(...)` / `_reset_bus_host(...)` / `_eh_reset_target(...)` | error recovery | `ErrorRecovery::*` |
| `megasas_get_seq_num(...)` / `_get_evt_log_info(...)` | event log read | `EvtLog::read` |
| `megasas_get_ld_list_from_firmware(...)` / `_get_pd_list_from_firmware(...)` | discovery | `Discovery::*` |
| `megasas_get_raid_map_*` | RAID map fetch | `RaidMap::fetch` |
| `megasas_aen_polling(...)` | AEN periodic poll | `Aen::tick` |
| `megasas_dump_*` | debug dumps | `Dump::*` |

## Compatibility contract

REQ-1: PCI ID table — every MegaRAID SAS silicon from SAS1078 through SAS3908 + Aero/Crusader. Frozen.

REQ-2: MFI cmd ABI — frozen against `mbox_defs.h`. Hundreds of opcodes for SCSI passthru, RAID mgmt, AEN, FW upgrade, log retrieval.

REQ-3: Fusion descriptor ABI — frozen against `megaraid_sas_fusion.h`. MPI-2-derived descriptors; per-IO-direction descriptor types.

REQ-4: Fast-Path discipline — driver-side LD lookup + physical-disk translation only when RAID map valid + LD not in transitional state. Falls back to FW path on any transitional state.

REQ-5: AEN — periodic poll for FW events (LD added, LD removed, PD failed, rebuild progress, foreign config, hot-spare assigned). Sequence-numbered for resync.

REQ-6: /dev/megaraid_sas chardev — major 252; ioctls for MFI passthru + adapter info. Frozen ABI.

REQ-7: FW image — `megaraid/*.fw` per silicon. Broadcom-signed.

REQ-8: SCSI mid-layer error recovery — abort → LUN reset → target reset → host reset.

REQ-9: Sysfs RAID configuration — RAID-level config via /sys/class/scsi_host/host*/ attrs; FW does the heavy lifting.

REQ-10: PD (Physical Disk) passthru — JBOD mode disks visible as SCSI passthru devices.

REQ-11: Foreign-config detection — disks moved from another HBA marked foreign; explicit user import via /dev/megaraid_sas.

REQ-12: Encryption (SED via TCG Opal) — supported on encryption-capable LDs; key mgmt via FW.

## Acceptance Criteria

- [ ] AC-1: PERC H730 (Dell, SAS3108) probes; FW reports version; `dmesg | grep megasas` matches upstream.
- [ ] AC-2: RAID-5 logical volume across 6 SAS disks presents; reads/writes match expected throughput.
- [ ] AC-3: Fast-Path: for RAID-0/1/10 LDs, driver bypasses FW; per-CPU IRQ distribution observable; latency lower than FW path.
- [ ] AC-4: storcli vendor tool: `storcli /c0 show all` succeeds via /dev/megaraid_sas.
- [ ] AC-5: PD passthru: JBOD disk visible as /dev/sdN; smartctl works.
- [ ] AC-6: AEN: pull a disk; AEN fires; LD marked degraded; FW initiates rebuild on spare.
- [ ] AC-7: Foreign config: move disks from another HBA; foreign-config flag; explicit import via storcli succeeds.
- [ ] AC-8: Error recovery: timeout escalation through abort → LUN reset → host reset.
- [ ] AC-9: FW upgrade via /dev/megaraid_sas chardev: signed image accepted; unsigned refused.
- [ ] AC-10: SED encryption: TCG Opal-capable LD; key install + lock/unlock cycle works.

## Architecture

**Two silicon families.** MFI silicon (pre-SAS-3) uses mailbox cmd transport — driver writes cmd into a DMA-mapped MFI frame, FW reads, executes, writes completion. Fusion silicon (SAS-3+) uses DMA-direct descriptors — driver writes a descriptor directly into a per-CPU reply queue; FW reads via DMA. MFI silicon is now mostly legacy; modern deployments are Fusion.

**MFI cmd transport.** MFI frame is a fixed-format DMA-mapped buffer with: header (opcode, status, length), payload (per-opcode union), sg-list. Driver allocates from a pool of `megasas_cmd` instances; each holds an MFI frame + tracking metadata. Sync issue blocks; async issue completes via IRQ.

**Fusion DMA-direct.** Per-CPU reply queue + per-CPU descriptor ring. Driver writes a 64-byte IO descriptor directly via MMIO write; FW DMAs in. Reply queue read via IRQ. Eliminates the mailbox round-trip latency.

**Fast-Path.** For RAID-0/1/5/6/10 LDs in steady state, the driver-side has all the info needed to compute the physical-disk SCSI cmd directly: striping geometry, parity location, mirror selection. `megaraid_sas_fp.c` does this translation. Fast-Path is taken only when:
- LD is online (not degraded, not rebuilding, not transitional).
- RAID map is fresh.
- IO size fits within one stripe.
- Otherwise: fall back to FW path (FW handles parity + mirroring).

**RAID map.** Driver caches a copy of the FW's RAID map (LD topology, stripe size, PD layout). Refreshed on AEN events affecting LD layout. Fast-Path consults the cached map.

**AEN polling.** Driver registers for AEN events via MFI cmd; FW posts events to a sequence-numbered queue; driver polls every N seconds, dequeues events, dispatches.

**/dev/megaraid_sas chardev.** Vendor tool entry point. `MFI_PASSTHRU` ioctl issues raw MFI cmd from userspace. Historical CVE source (CVE-2021-39685 missing CAP_SYS_RAWIO).

**Error recovery.** Standard SCSI escalation. Note: Fast-Path errors typically fall back to FW path automatically for retry; FW path errors escalate through the standard mechanism.

## Hardening

- MFI cmd pool sized at probe; over-alloc returns -EBUSY.
- Fusion descriptor count validated against FW caps.
- RAID map refresh atomic — fast-path readers see consistent map.
- AEN sequence-number tracked; missed events trigger full resync.
- /dev/megaraid_sas ioctl arg sizes validated per-opcode.
- Per-LD queue depth enforced.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — /dev/megaraid_sas ioctl ingress per-opcode size validation; bounded copy_from_user.
- **PAX_KERNEXEC** — scsi_host_template, MFI opcode dispatch, Fusion descriptor build, /dev/megaraid_sas ioctl dispatch, IRQ handler tables, AEN dispatcher, `pci_error_handlers` all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — /dev/megaraid_sas ioctl + sysfs entry paths per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `megasas_instance` refcount, per-MFI cmd refcount, per-Fusion-IO refcount, per-AEN subscriber refcount, RAID-map refcount saturating.
- **PAX_MEMORY_SANITIZE** — MFI frame DMA buffers cleared after submit (carry LBAs, sg-list addrs, scsi cmd data). Fusion descriptors cleared. /dev/megaraid_sas ioctl payload scratch cleared. SED encryption keys passed through FW never held in driver memory.
- **PAX_UDEREF** — SMAP/SMEP on /dev/megaraid_sas + sysfs.
- **PAX_RAP / kCFI** — scsi_host_template ops, MFI opcode dispatch, Fusion build dispatch, /dev/megaraid_sas ioctl dispatch, AEN handlers all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs/sysfs scrubbed; reveals MFI cmd pool + RAID map addresses + Fast-Path stripe maps.
- **GRKERNSEC_DMESG** — megasas verbose logs (AEN trace revealing LD topology, RAID map dumps, Fast-Path decisions) restricted from non-root.
- **CAP_SYS_RAWIO required on /dev/megaraid_sas** — closes CVE-2021-39685 missing-CAP class.
- **MFI passthru opcode allowlist + per-opcode size strict** — destructive opcodes (FW reset, FW write, LD delete, force-online-foreign-LD) require additional CAP_SYS_ADMIN. Closes CVE-2019-19528 stack-OOB class.
- **MFI passthru sg-list cap** — upstream's max per-cmd sg-list size enforced; closes 'unprivileged-with-CAP_SYS_RAWIO OOMs kernel via huge sg-list' class.
- **FW image signature mandatory** — Broadcom-signed; LOCKDOWN_INTEGRITY_MAX disables FW updates.
- **AEN payload validation** — per-event-type payload size validated; closes 'compromised FW posts garbage AEN' OOB.
- **AEN subscription rate-limit** — closes spam-subscribe DoS.
- **RAID map refresh atomic** — RAID map swap via RCU; Fast-Path readers see consistent snapshot; closes UAF on concurrent refresh + Fast-Path.
- **Fast-Path bounds-checked** — driver-computed physical-disk LBA + count validated against PD geometry before SCSI cmd issue. Closes a path where corrupt RAID map could cause Fast-Path to write to wrong disk / wrong LBA.
- **Foreign-config import gated** — importing foreign disks (which may contain other tenants' data) requires CAP_SYS_ADMIN.
- **SED key install gated** — TCG Opal key install via /dev/megaraid_sas requires CAP_SYS_ADMIN + LOCKDOWN bypass-disabled.
- **PD passthru per-disk permission scoped** — JBOD passthru commands respect cgroup-bdev / device-policy restrictions.
- **Host reset rate-limit** — 3 in 60s, then HBA offline.
- **FW core dump access** — CAP_SYS_ADMIN required.
- **Encryption-key memory hygiene** — when LDs are SED-encrypted, key material in ioctl payload kfree_sensitive immediately after FW install.
- **vendor-cmd-string ioctl input strictly NUL-terminated and length-bounded** — string-based vendor sub-commands have explicit length check + termination guarantee; closes parsing-bug class.

Per-doc rationale: megaraid_sas is the dominant enterprise RAID HBA. The /dev/megaraid_sas chardev has been the productive CVE source: CVE-2021-39685 (missing CAP_SYS_RAWIO), CVE-2019-19528 (stack OOB), CVE-2020-12114 (unsafe pointer in legacy ioctl). Defenses layer: CAP_SYS_RAWIO + opcode allowlist + per-opcode size + destructive-op CAP_SYS_ADMIN + sg-list cap close the chardev attack surface. Fast-Path adds a new attack surface (corrupt RAID map → wrong-disk write); bounds-check on driver-computed addresses closes it. Foreign-config import is the cross-tenant concern — disks moved from another HBA may contain other organizations' data; explicit CAP_SYS_ADMIN gate. The Rust translation's typed MFI cmd interface + RAID map RCU + Fast-Path bounds-check close the dominant historical bug classes structurally.

## Open Questions

- [ ] Q1: MFI silicon support — pre-SAS-3 silicon is mostly retired. Maintain or drop? Recommendation: maintain via separate sub-crate (mfi-legacy); deprecation warning at probe.
- [ ] Q2: Fast-Path correctness — driver-side parity computation has historically been tricky. Strong validation via verus + Kani.
- [ ] Q3: Tri-mode (NVMe behind controller) — modern silicon supports NVMe drives behind the RAID controller. Integration with nvme subsystem needs careful design (parallels mpt3sas Tri-mode but with RAID FW in front).
- [ ] Q4: storcli ABI compat — the dominant vendor tool. Verify our opcode allowlist + sizes don't break standard storcli operations.
- [ ] Q5: PERC-specific quirks (Dell) — Dell rebadges MegaRAID and adds PERC-specific behavior. Driver currently has Dell-specific paths; preserve.

## Verification

- **Kani SAFETY**: prove MFI cmd pool allocation cannot return a slot in use. Prove RAID map RCU read pairs with rcu_read_lock/unlock.
- **TLA+**: model the RAID map refresh + Fast-Path read concurrent paths; check Fast-Path always sees a complete map (no torn-read on refresh).
- **Verus**: functional spec of Fast-Path LBA translation — for any (LD lba, count) with valid RAID map, produces the correct (PD index, PD lba, count) tuples per RAID-0/1/5/6/10 geometry.
- **Kani+Verus**: invariant that every active MFI cmd has a tracked descriptor; every AEN subscription has a registered handler.
- **Integration**: PERC H740/H750 24-disk enclosure stress — RAID-6 + RAID-10 + JBOD mix, sustained IO 48h. Disk-pull/replace stress. Fast-Path vs FW-path A/B perf test. Foreign-config import smoke. SED encryption smoke.
- **Fuzz**: /dev/megaraid_sas ioctl fuzzer (MFI passthru mutations). RAID map fuzzer (mutate map fields → Fast-Path translator must refuse).
- **Penetration**: /dev/megaraid_sas open without CAP_SYS_RAWIO — refused. Destructive opcode without CAP_SYS_ADMIN — refused. Foreign-config import without CAP_SYS_ADMIN — refused.

## Out of Scope

- megaraid (legacy non-SAS PCI MegaRAID) — separate Tier-3 (low priority)
- mpt3sas (sibling SAS HBA, IT firmware) — covered in `mpt3sas.md`
- aacraid (sibling RAID HBA from Adaptec) — covered in `aacraid.md`
- arcmsr (Areca, separate RAID family) — separate Tier-3 (future)
- 3w-9xxx/3w-sas (3ware/AMCC, separate, mostly retired) — out of scope
