# Tier-3: drivers/scsi/aacraid/{linit,aachba,commsup,comminit,commctrl,dpcsup,src,rkt,rx,sa,nark}.c — Adaptec / Microsemi / Microchip AAC-RAID (PMC-Sierra RAID HBA — every Dell PowerEdge / HP ProLiant / Lenovo server with Adaptec RAID controller)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/scsi/00-overview.md
upstream-paths:
  - drivers/scsi/aacraid/linit.c                              (~3000 lines: Linux integration — SCSI host template, PCI lifecycle, sysfs)
  - drivers/scsi/aacraid/aachba.c                             (~3500 lines: HBA cmd translation — SCSI → AAC FIB)
  - drivers/scsi/aacraid/commsup.c                            (~2000 lines: FIB submission/completion — communication support layer)
  - drivers/scsi/aacraid/comminit.c                           (~1500 lines: HBA init — adapter version detect, IRQ setup, queue alloc)
  - drivers/scsi/aacraid/commctrl.c                           (~1300 lines: /dev/aac chardev ioctls + management API)
  - drivers/scsi/aacraid/dpcsup.c                             (~200 lines: deferred procedure call — async event processing)
  - drivers/scsi/aacraid/src.c                                (~1430 lines: SRCv2 silicon variant — modern SmartRAID 8000/9000)
  - drivers/scsi/aacraid/rkt.c                                (~120 lines: RKT silicon variant)
  - drivers/scsi/aacraid/rx.c                                 (~150 lines: RX silicon variant)
  - drivers/scsi/aacraid/sa.c                                 (~100 lines: SA silicon variant)
  - drivers/scsi/aacraid/nark.c                               (~70 lines: NARK silicon variant)
  - drivers/scsi/aacraid/aacraid.h                            (large header — types, FIB structures, opcodes)
  - include/scsi/scsi_*.h                                     (SCSI mid-layer)
-->

## Summary

aacraid is the driver for Adaptec (now Microsemi / Microchip after acquisitions) AAC-family RAID HBAs — the third major enterprise storage HBA family alongside LSI MegaRAID (megaraid_sas) and LSI mpt3sas. Adaptec RAID controllers ship by default in Dell PowerEdge (some generations), Lenovo ThinkServer, HP ProLiant (legacy), and many white-box server vendors. Covers Adaptec ASR/AAC silicon from the early 2010s (RX/RKT/SA) through the modern SmartRAID 8000/9000-series (SRCv2 silicon).

The driver provides:
- Hardware RAID: RAID-0, 1, 5, 6, 10, 50, 60. RAID metadata managed by firmware; driver presents resulting logical volumes to Linux as SCSI devices.
- JBOD passthrough: per-disk SCSI access when RAID is bypassed.
- Hot-spare and rebuild orchestration: FW-driven; events surfaced to userspace via netlink + sysfs + /dev/aac ioctl.
- Pluggable backplane management: enclosure services (SES) integration.
- Management ioctl interface `/dev/aac` for vendor tools (`arcconf`, `maxView Storage Manager`).
- Battery-backed cache + flash-backed cache for write-back consistency.

Source: ~13600 lines / 11 files. Compared to qla2xxx (83K) and lpfc (103K), aacraid is much smaller — the firmware does most of the work; driver is thin glue.

This Tier-3 covers all 11 files.

## Rust translation posture

aacraid is simpler than the FC HBA drivers because RAID is mostly FW-resident. Translation strategy:

- **`aacraid-core` crate** — PCI lifecycle, SCSI host template, FIB transport, ioctl, management.
- **Typed FIB (Firmware Interface Block).** Each FIB opcode is a typed struct. FIB layer is a request/response queue keyed on FIB handle (similar pattern to qla2xxx mailbox but lower opcode count).
- **Silicon-variant trait.** `SrcV2`, `Rkt`, `Rx`, `Sa`, `Nark` implementing an `AacSilicon` trait. Per-silicon: doorbell layout, IRQ setup, register offsets. Auto-detection at probe.
- **AacAdapter phase typing.** `AacAdapter<Probed → InitDone → Online>`.
- **/dev/aac chardev ioctls** — modest surface (~20 ioctls); each typed.
- **Logical-volume vs JBOD path** — typed `Lun<Logical | JBOD>` for SCSI cmd dispatch.

Grsec is mandatory. aacraid has had CVE history (CVE-2017-9059 ioctl OOB write, CVE-2013-2899 chardev permission, several FIB-handling bugs).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct aac_dev` | per-HBA state (silicon, FIBs, queues, raw_io_64) | `AacAdapter<Phase>` |
| `struct fib` | Firmware Interface Block | `Fib<Op>` (typed) |
| `struct aac_queue` | per-direction queue (host->FW or FW->host) | `AacQ<Direction>` |
| `aac_probe_one(pdev, ent)` / `_remove_one(pdev)` | PCI probe/remove | `AacAdapter::probe` / `Drop` |
| `aac_acquire_resources(dev)` / `_release_resources(dev)` | bus resources | `AacAdapter::resources` |
| `aac_send_command(dev, info, raw_io)` | FIB submit | `Fib::send` |
| `aac_queuecommand(host, scsi_cmd)` | SCSI cmd issue | `AacAdapter::queue_cmd` |
| `aac_intr_normal(dev, qid, isFastResponse, fib)` | IRQ completion | `Irq::on_completion` |
| `aac_command_normal(q)` / `_command_thread(dev)` | post-IRQ DPC | `Dpc::tick` |
| `aac_async_event(dev, aif_buffer)` | async event (AIF) | `Aif::on_event` |
| `aac_fib_alloc(dev)` / `_fib_init(...)` / `_fib_free(...)` | FIB pool | `FibPool::*` |
| `aac_init_adapter(dev)` / `_set_adapter_props(dev)` | post-probe init | `AacAdapter::init` |
| `aac_get_adapter_info(dev)` / `_get_container_info(...)` | discovery | `Discovery::*` |
| `aac_query_disk(dev, scsi_dev)` | per-disk attr query | `Disk::query` |
| `aac_eh_reset(...)` / `_eh_abort(...)` / `_eh_lun_reset(...)` | error recovery | `ErrorRecovery::*` |
| `aac_handle_aif(dev, fibctx, aif_buffer)` | AIF event handling | `Aif::handle` |
| `aac_open_chardev(...)` / `_release_chardev(...)` / `_ioctl(...)` | /dev/aac chardev | `Chardev::*` |
| `aac_send_passthru_ioctl(...)` | FIB passthru via ioctl | `Chardev::passthru` |
| `aac_get_pci_info(...)` / `_get_adapter_info_chardev(...)` | chardev info ioctls | `Chardev::info` |
| `aac_register_aif_notifier(...)` / `_unregister_aif_notifier(...)` | AIF subscriber registration | `AifNotifier::*` |
| `aac_src_init` / `_rkt_init` / `_rx_init` / `_sa_init` / `_nark_init` | per-silicon init | `AacSilicon::init` |
| `aac_src_select_comm` | per-silicon doorbell + queue setup | `AacSilicon::select_comm` |

## Compatibility contract

REQ-1: PCI ID table — every Adaptec ASR/AAC silicon: ASR-2405/2805 (Rx/RKT), ASR-71605/72405 (SrcV2 first-gen), ASR-8405/8805/81605 (SrcV2 second-gen, SmartRAID 8000), SmartRAID 9000 series + 3100/3200/3300/3400 (modern). Frozen.

REQ-2: FIB ABI — per-opcode request/response struct frozen against vendor spec. Hundreds of opcodes for SCSI passthru, RAID mgmt, enclosure mgmt, FW upgrade, log retrieval, etc.

REQ-3: Silicon variants — silicon-version detected at probe; per-variant code paths for doorbell layout + queue setup. Frozen against per-silicon register specs.

REQ-4: FW image — `aac/<silicon>.fw` per silicon variant. Adaptec-signed. PCI hardware secure-boot validates.

REQ-5: RAID volumes — FW-managed; driver presents resulting logical volumes as SCSI devices via `scsi_add_device`.

REQ-6: JBOD pass-through — per-disk SCSI passthrough when RAID is configured for JBOD mode on a disk.

REQ-7: Async Interface Block (AIF) — FW posts events (LUN added/removed, disk failed, rebuild progress, hot-spare activated, battery state) to a dedicated queue; driver polls + dispatches to userspace listeners.

REQ-8: /dev/aac chardev — major number 215; ioctls for FIB passthru + adapter info. Frozen ABI.

REQ-9: Battery-backed cache — driver reads battery state; surfaces in sysfs; raises events on low battery.

REQ-10: Enclosure services — SES discovery + slot mapping via SCSI INQUIRY + SES commands routed through FW.

REQ-11: Logical-volume hot-add — FW notifies via AIF; driver scans + adds LUN.

REQ-12: SCSI mid-layer error recovery — abort → LUN reset → host reset escalation.

## Acceptance Criteria

- [ ] AC-1: Adaptec SmartRAID 8805 probes; FW reports version; `dmesg | grep aacraid` matches upstream.
- [ ] AC-2: RAID-5 logical volume across 6 disks presents as `/dev/sda`; reads/writes work; throughput matches expected.
- [ ] AC-3: JBOD passthru: configure a disk in JBOD mode; per-disk `/dev/sdN` visible; smartctl works.
- [ ] AC-4: Hot-add: insert a disk; AIF event fires; controller rebuilds onto it; `arcconf` shows progress.
- [ ] AC-5: arcconf vendor tool: `arcconf getconfig 1` returns RAID config via /dev/aac.
- [ ] AC-6: Battery state: `cat /sys/.../battery_status` returns expected state on supported HBAs.
- [ ] AC-7: SES enclosure: `sg_ses --page=2 /dev/sg0` reports enclosure slot status.
- [ ] AC-8: Error recovery: timeout on a cmd → abort fires; if controller hangs → host reset; recovery completes.
- [ ] AC-9: FW upgrade via /dev/aac chardev: signed FW image accepted; unsigned refused.

## Architecture

**FIB transport.** Driver↔FW communication is FIB-based:
1. Allocate FIB from pool.
2. Fill in opcode + payload.
3. Queue to per-direction host-to-FW queue + ring doorbell.
4. FW executes + posts completion to FW-to-host queue + raises IRQ.
5. IRQ handler reads completion + dispatches to waiter (sync) or callback (async).

FIB pool sized at init from FW caps. Tagged FIB handle round-trips through FW.

**Per-silicon abstraction.** Silicon variants differ in: doorbell register layout, queue address mapping, IRQ source register, init sequence. `aacraid.h` has a `struct aac_driver_ident` per silicon; per-silicon init function (`aac_src_init`, etc.) sets up the right register offsets + doorbell. Rookery's `AacSilicon` trait makes the abstraction explicit.

**Discovery.** At init: `aac_get_adapter_info` returns adapter caps; `aac_get_container_info` returns logical-volume list. Driver iterates LUNs + adds SCSI devices. JBOD disks discovered via separate per-disk INQUIRY.

**AIF (async events).** Dedicated FIB queue for FW-initiated events. Events: LUN added, LUN removed, disk failed, rebuild started, rebuild progress, hot-spare activated, foreign-config detected, battery state change, enclosure event. Driver dispatches to per-process AIF subscribers (chardev fd subscribed via ioctl).

**/dev/aac chardev.** Vendor tool interface. Ioctls:
- `FSACTL_SENDFIB` / `_SEND_LARGE_FIB`: passthrough FIB (raw FW cmd).
- `FSACTL_OPEN_GET_ADAPTER_FIB`: AIF subscription.
- `FSACTL_QUERY_DISK`, `FSACTL_GET_PCI_INFO`: info.
- `FSACTL_FORCE_DELETE_DISK`: destructive (admin-only).
- `FSACTL_GET_VERSION_MATCHING`: version check.

Historical CVE area (CVE-2017-9059 OOB write on FIB passthru).

**Error recovery.** Standard SCSI escalation: cmd abort → LUN reset → controller reset → adapter offline.

## Hardening

- FIB pool exhaustion returns -EBUSY cleanly.
- FIB passthru via ioctl validates opcode + size before forwarding.
- AIF queue depth bounded.
- Per-LUN cmd timeout enforced; escalation rate-limited.
- Per-silicon init validated against expected register reads.
- FW image signature check via HBA secure-boot.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — /dev/aac ioctl ingress per-opcode size validation; bounded copy_from_user.
- **PAX_KERNEXEC** — scsi_host_template, FIB opcode dispatch, AIF handler, ioctl dispatch table, per-silicon op tables, pci_error_handlers all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — /dev/aac ioctl + sysfs entry paths per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `aac_dev` refcount, FIB refcount, AIF subscriber refcount saturating.
- **PAX_MEMORY_SANITIZE** — FIB DMA buffers cleared after submit. AIF event buffer cleared on subscriber unsubscribe. /dev/aac ioctl-payload scratch cleared.
- **PAX_UDEREF** — SMAP/SMEP on /dev/aac chardev + sysfs.
- **PAX_RAP / kCFI** — scsi_host_template ops, FIB opcode dispatch, AIF handlers, /dev/aac ioctl dispatch, per-silicon trait dispatch all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs/sysfs scrubbed; reveals FIB queue addresses + per-silicon register addresses.
- **GRKERNSEC_DMESG** — aacraid verbose logs (AIF event details revealing disk topology, FW state changes) restricted from non-root.
- **CAP_SYS_RAWIO required on /dev/aac** — closes historical CVE-2013-2899 class (chardev too permissive).
- **FIB passthrough opcode allowlist + per-opcode size strict** — destructive opcodes (force-delete-disk, FW write) require additional CAP_SYS_ADMIN. Closes CVE-2017-9059 OOB-write class.
- **FW image signature mandatory** — Adaptec-signed; LOCKDOWN_INTEGRITY_MAX disables.
- **AIF subscription rate-limit** — per-process AIF subscription rate-limited; closes "spam-subscribe to exhaust event queue" DoS.
- **Per-silicon op validation** — silicon-variant detection cross-checked against reg reads at init; mismatched detection refuses to bind. Closes "spoof silicon-id to take wrong code path" class.
- **Logical-volume isolation in multi-controller setup** — multi-controller server: AIF events scoped to the source controller; cross-controller event injection refused.
- **FORCE_DELETE_DISK strictly gated** — CAP_SYS_ADMIN + LOCKDOWN_INTEGRITY_MAX bypass-disabled. Destroys data; must require highest privilege.
- **Battery state read scoped** — battery-status reads CAP_SYS_ADMIN (could leak controller-failure-state info to unprivileged).
- **Host reset rate-limit** — 3 in 60s, then HBA offline.
- **JBOD passthru access scoped** — JBOD per-disk passthrough commands gated by render-of-block-device permissions (cgroup-bdev).
- **FW core dump access** — CAP_SYS_ADMIN required; may contain LUN data.
- **AIF event payload validated** — event header + per-event-type payload size validated; closes "compromised FW posts garbage AIF" OOB class.

Per-doc rationale: aacraid is a smaller driver than the FC HBAs but the /dev/aac chardev is a productive CVE source. The Rust translation's typed FIB + opcode allowlist + per-silicon trait dispatch close the historical bug classes. RAID storage controllers have a particular tenant-isolation concern: a misuse of force-delete-disk or FW-upgrade ops can destroy data on volumes the calling process should not have control over — hence the strict CAP_SYS_ADMIN + LOCKDOWN gating on destructive ops. Battery-state info leakage is a real concern in physical-host-leak threat models (a controller's failing battery is a maintenance window indicator).

## Open Questions

- [ ] Q1: Older silicon (Rx/RKT/SA/NARK) is end-of-life. Maintain or drop? Recommendation: maintain via the per-silicon trait — the per-variant init code is small (<200 lines each).
- [ ] Q2: Modern Microchip SmartRAID 3300+ silicon — new ABI? Verify upstream FIB ABI tracks.
- [ ] Q3: NVMe-attached disks behind aacraid — does SmartRAID 9000+ present NVMe drives? If yes, integration with nvme subsystem needs design.
- [ ] Q4: Hardware-encrypted-disk support (SED via TCG Opal) — driver-side support?

## Verification

- **Kani SAFETY**: prove FIB pool allocation cannot return a slot in use. Prove silicon-variant trait dispatch is exhaustive.
- **TLA+**: model the AIF subscriber + event dispatcher with concurrent subscribe/unsubscribe + event arrival; no UAF.
- **Verus**: functional spec of FIB passthru ioctl — for valid user FIB, dispatches to FW; for malformed, returns -EINVAL.
- **Kani+Verus**: invariant that every active AIF subscriber has a valid fctx; cleanup at chardev close removes all.
- **Integration**: SmartRAID 8805 stress — RAID-5 over 8 disks, sustained IO 24h; hot-add/-remove disk; rebuild stress; FW upgrade injection; vendor-tool (arcconf) full test matrix.
- **Fuzz**: /dev/aac ioctl fuzzer (FIB passthru with mutated opcode/payload); AIF event payload fuzzer.
- **Penetration**: /dev/aac open without CAP_SYS_RAWIO — refused. FORCE_DELETE_DISK without CAP_SYS_ADMIN — refused.

## Out of Scope

- Adaptec ASR-2010S/ASR-2120S (very old PCI cards) — out of scope
- megaraid_sas (LSI MegaRAID, sibling RAID HBA) — separate Tier-3
- mpt3sas (LSI SAS HBA) — covered in `mpt3sas.md`
- DPT i2o (older Adaptec, separate driver) — out of scope
- arcmsr (Areca RAID, separate driver) — separate Tier-3 (future)
