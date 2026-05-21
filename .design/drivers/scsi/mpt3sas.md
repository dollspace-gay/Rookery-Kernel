# Tier-3: drivers/scsi/mpt3sas/{mpt3sas_base,mpt3sas_scsih,mpt3sas_config,mpt3sas_ctl,mpt3sas_transport,mpt3sas_warpdrive,mpt3sas_trigger_diag}.c — LSI/Broadcom Fusion-MPT SAS-3 HBA + Tri-mode (every enterprise server with LSI/Avago/Broadcom storage controllers)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/scsi/00-overview.md
upstream-paths:
  - drivers/scsi/mpt3sas/mpt3sas_base.c               (~9040 lines: PCI lifecycle, MSI-X, reply-post free + reply-free queues, ioc state machine, request queue, doorbell, reset)
  - drivers/scsi/mpt3sas/mpt3sas_base.h
  - drivers/scsi/mpt3sas/mpt3sas_scsih.c              (~14180 lines: SCSI mid-layer integration, device discovery, expander/SAS topology, target/lun mgmt, error handling)
  - drivers/scsi/mpt3sas/mpt3sas_config.c             (~2715 lines: MPI config-page reads/writes — IOC, SAS, Persistent, BIOS, ...)
  - drivers/scsi/mpt3sas/mpt3sas_ctl.c                (~4520 lines: /dev/mpt3ctl chardev for vendor tools — passthrough, diag-buffer, BTDH, app_cmd)
  - drivers/scsi/mpt3sas/mpt3sas_transport.c          (~2195 lines: SAS transport class integration — phy/port/end-device/expander attributes)
  - drivers/scsi/mpt3sas/mpt3sas_trigger_diag.c       (~475 lines: Diag-trigger captures — master/event/scsi/MPI on-fault auto-collect)
  - drivers/scsi/mpt3sas/mpt3sas_warpdrive.c          (~300 lines: WarpDrive SSD acceleration)
  - drivers/scsi/mpt3sas/mpt3sas_debugfs.c
  - drivers/scsi/mpt3sas/mpi/                          (firmware ABI — MPI 2.5/2.6 spec headers, ~2000 lines)
  - include/scsi/scsi_*.h                              (SCSI mid-layer interfaces)
  - drivers/scsi/scsi_transport_sas.c                  (SAS transport class — separate Tier-3 dependency)
-->

## Summary

mpt3sas is the driver for LSI/Avago/Broadcom Fusion-MPT SAS-3 host bus adapters (HBAs) — every enterprise server built in the last decade has at least one of these silicon variants either as a discrete HBA card (the SAS3008/3408/3416/3508/3516 controllers) or as a mezzanine on the motherboard. Includes Tri-mode controllers (SAS3508/3516/3616/3808/3816) that can drive SAS, SATA, **and** PCIe NVMe drives behind a single HBA — the architecture that powers most modern NVMe-of-Fabrics-to-local-SSD enterprise storage chassis. Also covers the older SAS2 silicon via the sister `mpt2sas` driver (Rookery would unify both behind a single Rust translation since the MPI ABI is largely shared).

The driver is the largest single SCSI driver in upstream Linux at ~33500 lines of C, second only to mlx5 in driver-tree size. Why so big: it implements:

- Full MPI-2.5/2.6 firmware ABI (a TLV-style request/response protocol with hundreds of opcodes covering SCSI IO, SAS topology mgmt, config pages, diag, FW upgrade).
- SCSI mid-layer integration (every block-device IO to a SAS/SATA disk goes through here).
- SAS transport class integration (phy/port/end-device/expander as kernel objects with sysfs).
- Device discovery state machine (handles expander cascades, multi-pathing, hot-plug, NVMe-behind-HBA enumeration).
- Error recovery (per-LUN abort, target reset, host reset, escalation pyramid).
- Vendor-tool chardev `/dev/mpt3ctl` (Broadcom's `storcli` and `sas3ircu` issue arbitrary MPI commands through here — large attack surface).
- Diag-trigger autocapture (on-fault FW-side trace collection for post-mortem).

This Tier-3 covers ~33500 lines.

## Rust translation posture

mpt3sas is gnarly — the size is dominated by the SCSI device discovery + SAS topology + error recovery state machines. Rust translation strategy:

- **MPI ABI as typed.** Replace `mpi/mpi2_*.h` enum-based message structs with typed `bilge` bit-packed structs. Each MPI opcode is a method on `MpiIoc`: `ioc.config_page_read(page_type, ...)` rather than `mpt3sas_config_get_*` with macros.
- **Reply-post / reply-free queue pair.** mpt3sas uses two distinct queues: reply-post (firmware writes completions here, driver reads) + reply-free (driver writes free reply buffers here for firmware to consume). Rust types: `ReplyPostQ` (read-only-for-driver) + `ReplyFreeQ` (write-only-for-driver) with DMA-direction enforced.
- **IOC (I/O Controller) state machine.** `Ioc<Phase>` typed lifecycle: `Reset → Ready → Operational → Coredump → FaultReset`. Most operations only permitted in `Operational`.
- **SAS topology tree.** Replace `_scsih_sas_host_*` list-of-lists with a typed `SasTopology { root: SasHost, expanders: BTreeMap<Sas Addr, Expander>, devices: BTreeMap<SasAddr, EndDevice> }`. Insert/remove via typed events.
- **Device discovery state machine.** The `_scsih_topology_change_event` handler in upstream is ~2000 lines of nested switch/case. Rust: a typed `enum TopologyEvent { DeviceAdded, DeviceRemoved, ExpanderAdded, ExpanderRemoved, PhyStatusChange, ... }` dispatched through pattern matching.
- **Error recovery escalation.** Per-LUN abort → target reset → host reset is a state machine; promote to typed.
- **/dev/mpt3ctl ioctls.** Largest userspace attack surface; mandatory CAP_SYS_ADMIN; per-ioctl typed structs (no `u8 *` blobs).
- **NVMe-behind-HBA passthrough.** Tri-mode controllers expose NVMe drives as PCIe-passthrough; driver maintains shadow state for namespaces.
- **Diag-trigger logic.** Typed event triggers with a fixed allowlist.

The grsec/PaX section is mandatory: mpt3sas has had multiple CVEs (CVE-2019-19073 `_scsih_pcie_topology_change_event_debug` OOB, CVE-2021-39636 ioctl missing capability check, several MPI-parsing bugs). The /dev/mpt3ctl chardev is the largest surface and historically the most-bug-prone.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct MPT3SAS_ADAPTER` | per-HBA controller state (IOC, queues, topology, scsi_host, /dev/mpt3ctl) | `Mpt3Adapter<Phase>` |
| `struct mpt3sas_facts` | IOC capability facts (queue counts, msg sizes, etc.) | `IocFacts` |
| `mpt3sas_base_attach(ioc)` / `_base_detach(ioc)` | IOC bring-up / tear-down | `Ioc::attach` / `Drop` |
| `_base_diag_reset(ioc)` / `_base_make_ioc_operational(ioc)` | reset / make operational | `Ioc::reset` / `_make_operational` |
| `mpt3sas_base_get_smid(ioc, cb_idx)` / `_get_smid_scsiio(...)` / `_get_smid_hpr(...)` | allocate System Message ID (request slot) | `RequestPool::alloc_smid` |
| `mpt3sas_base_free_smid(ioc, smid)` | free SMID | `RequestPool::free_smid` |
| `_base_put_smid_default(...)` / `_put_smid_scsi_io_atomic(...)` / `_put_smid_fast_path(...)` | submit request (different queue paths) | `RequestQ::submit` |
| `_base_interrupt(irq, data)` / `_base_msix_enable(ioc)` | IRQ handler / MSI-X | `Ioc::on_irq` / `_enable_msix` |
| `_scsih_probe(pdev, ent)` / `_scsih_remove(pdev)` | PCI probe / remove | `Mpt3Adapter::probe` / `Drop` |
| `_scsih_queue_rescan(ioc)` | trigger rescan | `Discovery::queue_rescan` |
| `_scsih_sas_topology_change_event(ioc, event)` | SAS topology event handler | `Discovery::on_topology_event` |
| `_scsih_sas_device_add(ioc, sas_device)` / `_remove(...)` | add/remove SAS end-device | `Discovery::add_sas_device` |
| `_scsih_expander_add(ioc, handle)` / `_remove(...)` | SAS expander add/remove | `Discovery::add_expander` |
| `_scsih_pcie_topology_change_event(ioc, event)` | NVMe-behind-HBA topology event | `Discovery::on_pcie_topology_event` |
| `_scsih_qcmd(scsi_host, scmd)` | queue scsi_cmd to firmware | `ScsiHost::queue_cmd` |
| `_scsih_io_done(ioc, smid, msix_index, reply)` | IO completion | `Ioc::io_completion` |
| `_scsih_abort(scmd)` / `_dev_reset(scmd)` / `_target_reset(scmd)` / `_host_reset(scmd)` | error recovery | `ErrorRecovery::abort` etc. |
| `mpt3sas_config_get_*(ioc, ...)` / `_config_set_*(...)` | MPI config page R/W | `Config::get_*` / `set_*` |
| `mpt3sas_transport_*` | SAS transport class hooks | `SasTransport::*` |
| `_ctl_ioctl(file, cmd, arg)` | /dev/mpt3ctl ioctl | `MptCtl::ioctl` |
| `_ctl_do_mpt_command(ioc, ioctl_header, mf)` | passthrough MPI cmd from userspace | `MptCtl::passthrough` |
| `_ctl_diag_register(ioc, diag_register)` / `_diag_unregister(...)` / `_diag_query(...)` / `_diag_release(...)` / `_diag_read_buffer(...)` | diag buffer ioctls | `MptCtl::diag_*` |
| `mpt3sas_trigger_master(...)` / `_event(...)` / `_scsi(...)` / `_mpi(...)` | diag trigger from various events | `DiagTrigger::*` |
| `_scsih_pcie_device_add_to_list(ioc, pcie_device)` | NVMe behind HBA add | `Discovery::add_pcie_device` |
| `mpt3sas_warpdrive_setup(ioc)` | WarpDrive SSD setup | `WarpDrive::setup` |

## Compatibility contract

REQ-1: PCI ID table — every LSI/Avago/Broadcom SAS-3 controller: SAS3008/3108/3216/3224/3308/3316/3408/3416/3508/3516/3608/3616/3716/3808/3816 + Aero/Sea Tri-mode IDs. Frozen.

REQ-2: MPI 2.5/2.6 ABI — frozen against `drivers/scsi/mpt3sas/mpi/mpi2*.h`. All request opcodes (`MPI2_FUNCTION_*`) + IOC events (`MPI2_EVENT_*`) supported.

REQ-3: Reply queue pair — reply-post queue (firmware-write, driver-read) + reply-free queue (driver-write, firmware-read). Sizes derived from `facts.max_reply_descriptor_post_queue_depth`. Each MSI-X vector has its own reply-post queue (for parallelism).

REQ-4: Request pool (SMID-indexed) — fixed pool of System Message IDs allocated at probe; each in-flight cmd reserves an SMID. Per-SMID: a request frame + a chain frame (for sg-list extension). Pool sizes derived from `facts.request_credit`.

REQ-5: SCSI mid-layer integration — host template (`scsi_host_template`) registered; `queuecommand` is the per-SCSI-cmd entry point.

REQ-6: SAS transport class — `_transport.c` registers per-phy/port/end-device/expander sysfs attributes; matches userspace `sg_*` and `smp_*` tooling expectations.

REQ-7: Device discovery — IOC events (`MPI2_EVENT_SAS_TOPOLOGY_CHANGE_LIST`) processed; new SAS end-devices/expanders added to topology tree + sysfs; removed devices cleaned up.

REQ-8: NVMe-behind-HBA — Tri-mode controllers expose NVMe drives through MPI; driver maintains shadow PCIe-device state + handles topology events.

REQ-9: Error recovery — 4-level escalation: per-LUN abort (`MPI2_FUNCTION_SCSI_TASK_MGMT` TASK_ABORT) → target reset (TARGET_RESET) → host reset (controller-wide reset) → adapter offline.

REQ-10: /dev/mpt3ctl chardev — vendor tools (storcli, sas3ircu, sas3flash) issue passthrough MPI commands + diag-buffer ops. Frozen ioctl ABI (`MPT3COMMAND`, `MPT3IOCINFO`, `MPT3DIAGREGISTER`, `MPT3DIAGRELEASE`, `MPT3DIAGREADBUFFER`, `MPT3DIAGUNREGISTER`, `MPT3DIAGQUERY`).

REQ-11: Diag-triggers — on-fault auto-capture: master triggers (`mpt3sas_trigger_master`) for high-level events; event triggers for specific IOC events; SCSI triggers for sense-data patterns; MPI triggers for command failures.

REQ-12: Multi-pathing — driver hands devices to scsi_dh; multipath-tools manages the per-path failover.

REQ-13: FW upgrade — via `/dev/mpt3ctl` `MPT3COMMAND` with `MPI2_FUNCTION_FW_DOWNLOAD`; signature check at FW level.

REQ-14: SR-IOV — some Tri-mode controllers expose VFs. Limited support; not commonly deployed.

## Acceptance Criteria

- [ ] AC-1: `lspci` shows LSI SAS3008; Rookery probes; `dmesg | grep mpt3sas` shows expected init log.
- [ ] AC-2: `lsblk` shows attached SAS/SATA drives with vendor/model from SCSI INQUIRY; reads/writes work; smartctl reports SMART data.
- [ ] AC-3: Tri-mode: NVMe drive in the same chassis shows up as `nvme0n1` enumerated via mpt3sas; bandwidth on the NVMe matches PCIe Gen4 line rate.
- [ ] AC-4: SAS expander cascade: 24-drive shelf via expander; all 24 drives enumerate with consistent /dev/sd* mapping; `sg_map` shows correct topology.
- [ ] AC-5: Hot-plug: insert drive into JBOD slot; topology-change event fires; new /dev/sdN appears; remove → device cleanly drops.
- [ ] AC-6: Error recovery: simulate timeout on a cmd → abort fires; if abort fails, target-reset fires; if target-reset fails, host-reset; recovery completes within 60s.
- [ ] AC-7: /dev/mpt3ctl: `storcli /c0 show` (Broadcom tool) succeeds; reads adapter info via MPT3IOCINFO + MPT3COMMAND passthrough.
- [ ] AC-8: Diag buffer: `mpt3ctl-diag-trigger` registers a buffer; trigger an event; buffer captured; readable via MPT3DIAGREADBUFFER.
- [ ] AC-9: FW upgrade smoke: small test FW image (signed) loaded via MPT3COMMAND FW_DOWNLOAD; controller reboots cleanly.
- [ ] AC-10: Concurrent IO at scale: fio mixed RW 4K randread+randwrite 32-queue-depth across 16 SAS SSDs sustains controller's spec'd IOPS (4M+ for SAS3 12 Gb/s × multi-lane).

## Architecture

**IOC state machine.** Upstream's `_base_make_ioc_ready` walks through: HARD_RESET → IOC_FACTS → PORT_FACTS → ENABLE_PORT → wait for OPERATIONAL → IOC_INIT → enable EVENTS. Rookery `Ioc<Phase>` types this:

```rust
impl Ioc<Reset> {
    pub fn facts(self) -> Result<Ioc<HaveFacts>, IocError>;
}
impl Ioc<HaveFacts> {
    pub fn init(self) -> Result<Ioc<Operational>, IocError>;
}
```

**Request pool.** SMIDs are indices into a fixed-size pool of request frames + chain frames. Allocation via `RequestPool::alloc_smid(cb_idx)` returns a typed `Smid<T>` where `T` distinguishes the callback path (SCSI IO vs configuration vs HPR — high-priority-request). Free via `Drop` on the Smid. The cb_idx-to-callback-fn dispatch is a kCFI-signed table.

**Reply queue.** Each MSI-X vector has its own reply-post queue (DMA-mapped, FW-write, driver-read). The reply-free queue is shared. NAPI-style poll on each reply-post queue per IRQ: walk completions, dispatch to the SMID's callback, advance reply-post head, advance reply-free with the freed reply frame.

**SCSI mid-layer integration.** `_scsih_qcmd` takes a `scsi_cmnd`, allocates an SMID, fills the MPI SCSI_IO_REQUEST, submits via the per-vector request queue. Completion in the per-vector IRQ handler walks `_scsih_io_done` which translates MPI status → `scsi_cmnd->result` and calls `scmd->scsi_done`.

**Device discovery.** IOC events drive topology updates. The `_scsih_sas_topology_change_event` walks the event's per-phy state array (added/removed/no-change for each phy); for added phys, issues a `MPI2_FUNCTION_SAS_GET_DEVICE_INFO` to learn the device's SAS address + type; for removed phys, walks the topology tree and removes the device. New SAS end-devices get `_scsih_sas_device_add_to_list` + sysfs registration + SCSI-host slave-alloc → SCSI device enumerated.

**SAS topology tree.** Replaces upstream's `sas_device_list` / `expander_list` / `pcie_device_list` with a typed `SasTopology` keyed on SAS address. Operations: `insert_device`, `remove_device`, `find_by_handle`, `find_by_address`. Holds an `Arc<EndDevice>` per device; SCSI mid-layer holds another reference; sysfs holds a third — three-way refcount managed by `Drop`.

**Error recovery escalation.** When an in-flight cmd times out:
1. `_scsih_abort` issues `MPI2_FUNCTION_SCSI_TASK_MGMT` with `TASK_ABORT`. If success, done.
2. If abort fails, `_scsih_dev_reset` issues TASK_MGMT with `LUN_RESET`. If success, done.
3. If LUN reset fails, `_scsih_target_reset` issues TASK_MGMT with `TARGET_RESET`. If success, done.
4. If target reset fails, `_scsih_host_reset` issues a controller-wide reset; all in-flight cmds returned with `DID_RESET`. If host reset fails, adapter goes offline.

Rookery models this as a `RecoveryLevel` enum with explicit transitions.

**/dev/mpt3ctl ioctls.** Vendor tool entry point. Major ioctls:
- `MPT3IOCINFO` — returns adapter info (frame size, queue depths, FW version).
- `MPT3COMMAND` — issue an arbitrary MPI command from userspace. Up to 16 MB sg-list per direction.
- `MPT3DIAGREGISTER` / `MPT3DIAGRELEASE` / `MPT3DIAGREADBUFFER` / `MPT3DIAGUNREGISTER` / `MPT3DIAGQUERY` — register a host-RAM buffer for FW to write diag traces into.

This is the largest userspace attack surface. Historical CVEs concentrate here (CVE-2021-39636 missing CAP_SYS_RAWIO check).

**Diag triggers.** Configurable per-event auto-capture. Master triggers fire on driver-defined high-level events (FAULT, IOC_RESET, etc.); event triggers fire on specific MPI2_EVENT_* values; SCSI triggers on sense-key patterns; MPI triggers on command-failure status. Each trigger captures a registered diag buffer's contents to a userspace tool.

## Hardening

- SMID allocations bounded by `facts.request_credit`; over-alloc returns ENOMEM cleanly.
- All MPI response IOCStatus codes inspected; non-success returns appropriate -errno.
- Chain-frame chains bounded; oversized sg-lists return -EINVAL.
- Per-LUN abort timeout (default 30s) enforced; escalation strict.
- Host-reset is the last resort; cannot retry endlessly.
- /dev/mpt3ctl MPT3COMMAND validates user MPI request frame size against expected per-opcode size.
- /dev/mpt3ctl MPT3COMMAND sg-list allocations bounded (16 MB upstream).
- Diag-buffer registration enforces buffer-size cap.
- Topology event payload size validated.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — `/dev/mpt3ctl` ioctl ingress is the primary user surface; every ioctl arg validated against expected size before copy_from_user; sg-list allocations bounded.
- **PAX_KERNEXEC** — `scsi_host_template`, ioctl dispatch table, IRQ handler, MPI cb_idx callback table, SAS transport ops, `pci_error_handlers` all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — /dev/mpt3ctl ioctl, sysfs, sg-ioctl, scsi-generic entry paths are per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `MPT3SAS_ADAPTER` refcount, per-SAS-device refcount, per-expander refcount, per-PCIe-device refcount, per-request-SMID refcount use saturating refcounts; underflow at remove is a hard panic.
- **PAX_MEMORY_SANITIZE** — request frames + chain frames + reply frames `kfree_sensitive` (carry SCSI cmd data, LBAs, possibly tenant data). Per-SAS-device context cleared on free. Diag buffers cleared at release.
- **PAX_UDEREF** — SMAP/SMEP on all /dev/mpt3ctl entry points + sysfs + sg-ioctl.
- **PAX_RAP / kCFI** — `scsi_host_template` ops, `_scsih_io_done` callback, SAS transport ops, ioctl dispatch table, diag-trigger handlers all kCFI-signed.
- **GRKERNSEC_HIDESYM** — sysfs/debugfs dumps that expose internal pointers (request frame addresses, SMID-to-cmd map, topology tree) scrubbed for non-root.
- **GRKERNSEC_DMESG** — verbose mpt3sas log messages (discovery progress, IO failure with cmd contents) restricted from non-root.
- **CAP_SYS_RAWIO required on /dev/mpt3ctl** — every ioctl on the chardev. Closes historical CVE-2021-39636 (missing CAP_SYS_RAWIO on a path that issued raw MPI commands).
- **CAP_SYS_ADMIN required for diag-buffer ops** — diag buffers can contain in-flight SCSI cmd data including kernel buffer addresses; must not be exposed to unprivileged.
- **MPT3COMMAND opcode allowlist when not root** — even with CAP_SYS_RAWIO, certain destructive MPI opcodes (`MPI2_FUNCTION_RAID_ACTION` for destructive RAID ops, `MPI2_FUNCTION_FW_DOWNLOAD` for FW flash, `MPI2_FUNCTION_FORMAT`-equivalent ops) require additional CAP_SYS_ADMIN.
- **FW image signature mandatory** — `MPI2_FUNCTION_FW_DOWNLOAD` path verifies the Broadcom-signed FW manifest before issuing; rejects unsigned FW images even with CAP_SYS_ADMIN. Lockdown mode integration: when LOCKDOWN_INTEGRITY_MAX, FW flash refused entirely.
- **MPT3COMMAND sg-list cap strict** — upstream's 16 MB cap is enforced; an unprivileged-with-CAP_SYS_RAWIO process cannot OOM the kernel via giant sg-lists.
- **Diag-buffer size cap strict** — per-buffer size capped (typically 1 GB per upstream); buffer count per IOC capped.
- **Topology event payload validation** — `MPI2_EVENT_SAS_TOPOLOGY_CHANGE_LIST` per-phy entry count validated against expected; closes parser OOB class (CVE-2019-19073 historical).
- **SAS-address bounds check** — sas_address is u64; lookups validated against the topology tree before dereferencing. A compromised IOC sending forged sas_addresses in events cannot trigger UAF.
- **Reply-post queue head tracked atomically** — concurrent IRQ + thread cleanup paths cannot double-process a reply (closes a use-after-free class).
- **Error-recovery escalation rate-limited** — host-reset capped to 3 per 60s; further triggers permanent-down rather than infinite-retry. Closes "compromised FW gaslights us into endless reset" class.
- **NVMe-behind-HBA passthrough scoped** — Tri-mode NVMe drives are presented through mpt3sas; admin-passthrough (NVMe ADMIN_CMD) requires CAP_SYS_ADMIN even though normal IO works as standard SCSI.
- **WarpDrive admin gate** — `/dev/mpt3ctl`-issued WarpDrive ops require CAP_SYS_ADMIN.
- **Crash-dump access gated** — diag-trigger-captured buffers contain in-flight cmd data; reader requires CAP_SYS_ADMIN.

Per-doc rationale: mpt3sas is the most-deployed enterprise storage HBA driver. Its size (33500 lines) and ABI surface (MPI ~2000 lines of headers, /dev/mpt3ctl ioctls) make it a productive bug-hunting target. Historical CVEs (CVE-2019-19073 topology-event OOB, CVE-2021-39636 missing CAP check) demonstrate the bug class. The /dev/mpt3ctl chardev is the single most-attacked path because it exposes raw MPI command issuance to userspace; CAP_SYS_RAWIO requirement + opcode-allowlist for the most destructive ops + FW signature mandatory close the major classes. The Rust translation's typed Smid<T> + typed MPI ABI + typed Ioc<Phase> close additional classes structurally. SAS topology tree as typed structure (rather than upstream's lists-of-lists) makes consistency invariants easier to enforce.

## Open Questions

- [ ] Q1: Unify mpt2sas + mpt3sas under one Rust translation? The MPI ABI is largely shared (MPI 2.0/2.5/2.6 differ in opcodes added per generation). Recommendation: unify behind a `MpiAbiVersion` enum that gates per-version opcodes; trade-off is one larger crate vs two duplicated.
- [ ] Q2: WarpDrive (SSD-acceleration) is a legacy feature. Should Rookery support it or punt as out-of-scope? Recommendation: punt — minimal modern deployment.
- [ ] Q3: SR-IOV on Tri-mode controllers — limited deployment. Punt to a follow-up?
- [ ] Q4: NVMe-behind-HBA shadow state — driver maintains an in-memory shadow of each NVMe namespace's identify data. Should this be synchronized with the kernel's nvme subsystem (so `nvme list` works correctly) or kept private to mpt3sas? Upstream is private + shadow; same plan.
- [ ] Q5: storcli ABI compatibility — Broadcom's `storcli` is the primary userspace tool; its ABI to /dev/mpt3ctl is documented but not formally specified. Question: should Rookery's strict opcode allowlist break any storcli ops? Run the full storcli test matrix.
- [ ] Q6: Diag-trigger rules — upstream allows arbitrary trigger config via /dev/mpt3ctl. Rookery could tighten with a fixed allowlist of triggerable events. Trade-off: lose flexibility for vendor support tools.

## Verification

- **Kani SAFETY**: prove `Smid<T>` cannot be used after free (typestate). Prove SMID allocator cannot return a slot already in use. Prove sg-list allocations bounded.
- **TLA+**: model the IOC state machine with concurrent reset + IO completion + topology events. Check no IO can complete during a reset (would crash); check all in-flight IOs are returned with DID_RESET after host reset.
- **Verus**: functional spec of `_scsih_qcmd` — given a valid scsi_cmnd, produces an MPI SCSI_IO_REQUEST with matching LBA + xfer_length + flags + sg-list; returns either submitted or SCSI_MLQUEUE_HOST_BUSY (no other path).
- **Kani+Verus**: invariant that every submitted SMID has a corresponding scsi_cmnd tracking entry; every completion's SMID is in the tracking table; no leaks.
- **Integration**: full enterprise storage smoke — 24-drive JBOD via SAS expander, mixed SAS-HDD + SAS-SSD + SATA-SSD + NVMe Tri-mode; fio workload sustained 24h with no errors; hot-plug 100 cycles. Error injection: random IO timeouts → escalation works. FW reset → recovery.
- **Fuzz**: /dev/mpt3ctl ioctl fuzzer (MPT3COMMAND with mutated MPI requests); should always return -EINVAL or -EPERM, never panic. Diag-buffer fuzzer (register/release races).
- **Penetration**: unprivileged user attempts /dev/mpt3ctl — refused; user with CAP_SYS_RAWIO attempts destructive MPI opcode — refused without additional CAP_SYS_ADMIN; user with both caps but no LOCKDOWN bypass attempts unsigned FW flash — refused.

## Out of Scope

- mpt2sas (SAS-2 sibling) — separate Tier-3, or unified via Q1
- mptlan (LAN-over-SAS, legacy) — out of scope
- megaraid (separate LSI driver for MegaRAID controllers) — separate Tier-3
- aacraid (Adaptec, separate driver) — separate Tier-3
- SAS transport class (`scsi_transport_sas`) — separate Tier-3 dependency
- SCSI mid-layer (`drivers/scsi/scsi_*.c`) — covered in `drivers/scsi/` core docs
- block layer multi-queue (`block/blk-mq*`) — covered in `block/` Tier-3
- WarpDrive — punt per Q2
- SmartRAID NVMe-of-the-future variants — future iteration
