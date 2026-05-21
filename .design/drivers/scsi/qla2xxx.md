# Tier-3: drivers/scsi/qla2xxx/{qla_os,qla_init,qla_mbx,qla_isr,qla_iocb,qla_target,qla_attr,qla_bsg,qla_gs,qla_edif,qla_nvme,qla_mid,qla_sup,qla_nx,qla_mr,qla_tmpl,qla_dbg,qla_dfs}.c — QLogic Fibre Channel HBA (every enterprise SAN deployment, FC + FCoE + NVMe-FC)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/scsi/00-overview.md
upstream-paths:
  - drivers/scsi/qla2xxx/qla_os.c                            (~7000 lines: PCI lifecycle, SCSI host template, error recovery, sysfs glue)
  - drivers/scsi/qla2xxx/qla_init.c                          (~6000 lines: HBA init, login flow, port discovery, RSCN handling, async events)
  - drivers/scsi/qla2xxx/qla_mbx.c                           (~7260 lines: mailbox commands — primary HBA control interface, hundreds of opcodes)
  - drivers/scsi/qla2xxx/qla_isr.c                           (~4810 lines: IRQ handling, async events, IOCB completion)
  - drivers/scsi/qla2xxx/qla_iocb.c                          (~5000 lines: IOCB build + send — request queue protocol)
  - drivers/scsi/qla2xxx/qla_target.c                        (~7000 lines: SCSI target mode — qlt_* — used by LIO target stack)
  - drivers/scsi/qla2xxx/qla_attr.c                          (~3000 lines: sysfs + bsg attrs)
  - drivers/scsi/qla2xxx/qla_bsg.c                           (~3000 lines: BSG passthrough for vendor tools)
  - drivers/scsi/qla2xxx/qla_gs.c                            (~5000 lines: FC GS — Name Server / Management Server queries)
  - drivers/scsi/qla2xxx/qla_edif.c                          (FC-SP-2 / EDIF — encryption + authentication)
  - drivers/scsi/qla2xxx/qla_nvme.c                          (~1330 lines: NVMe-over-FC integration with nvme-core)
  - drivers/scsi/qla2xxx/qla_mid.c                           (NPIV — N_Port ID Virtualization, multi-WWPN per port)
  - drivers/scsi/qla2xxx/qla_sup.c                           (NVRAM / flash / serdes utilities)
  - drivers/scsi/qla2xxx/qla_nx.c, qla_nx2.c                 (8030/8031 — legacy iSCSI variants, mostly out-of-scope)
  - drivers/scsi/qla2xxx/qla_mr.c                            (FCoE bridge support)
  - drivers/scsi/qla2xxx/qla_tmpl.c                          (~1100 lines: hardware extended-loop-init template)
  - drivers/scsi/qla2xxx/qla_dbg.c, qla_dfs.c                (debug, debugfs)
  - drivers/scsi/qla2xxx/qla_def.h                          (huge — ~7000 lines of types)
  - include/scsi/scsi_*.h                                    (SCSI mid-layer)
  - drivers/scsi/scsi_transport_fc.c                         (FC transport class — separate Tier-3 dependency)
  - drivers/nvme/host/fc.c                                   (NVMe-FC transport — sits on top via nvme_fc API)
-->

## Summary

qla2xxx is the driver for QLogic (now Marvell) Fibre Channel HBAs — the dominant FC HBA family in enterprise SAN, second only to Emulex/lpfc. Covers QLE25xx (8 Gbps), QLE26xx (16 Gbps), QLE27xx (32 Gbps), QLE28xx (64 Gbps), and the high-end QLE2870 (128 Gbps) silicon. Deployed in every storage-array-attached Linux server in finance, healthcare, manufacturing, government, telco. Supports both initiator mode (host-side, talking to SAN-attached storage arrays) and target mode (server presenting LUNs over FC, used in LIO target stack for fc-target appliances).

Modern features:
- **NVMe-over-FC** — NVMe SSD storage over FC fabric (replacing FCP-SCSI for low-latency workloads). qla2xxx integrates with nvme-core.
- **FC-SP-2 / EDIF** — Encryption at the FC layer (DH-CHAP authentication + AES-GCM encryption). For HIPAA/PCI-compliant SANs.
- **NPIV** — N_Port ID Virtualization, multiple WWPNs (World Wide Port Names) per physical port, used for hypervisor-side FC virtualization.
- **64/128 Gbps** — current-gen silicon line rates; saturating at 2 × 64 Gbps + RDMA-equivalent latency.

The driver is huge: ~83000 lines / 20 files. Largest is qla_mbx.c (~7260 lines, hundreds of mailbox-command implementations).

This Tier-3 covers the full qla2xxx driver.

## Rust translation posture

qla2xxx has a long history of bugs in three areas: (1) mailbox command construction (hundreds of opcode-specific format functions in qla_mbx.c, each can mis-format), (2) target-mode state machine (qla_target.c is a complex per-port state machine with many edge cases), (3) BSG passthrough (qla_bsg.c — passes user-supplied buffers to the FW). Rust translation:

- **`qla2xxx-core` crate** — PCI lifecycle, init, IOCB request/response, mailbox cmd transport, IRQ handling.
- **Typed mailbox cmd interface.** Each MBOX opcode becomes a method: `mbx.set_target_parameters(target_id, params) -> Result<...>`. The hundreds of opcodes group naturally into sub-modules (init, port-mgmt, fcp, nvme, mgmt). `bilge` for bit-packed structs.
- **HBA phase typing.** `Qla<Probed → Initialized → LoggedIn → Online>`.
- **IOCB request/response queue.** Typed `RequestQ` (write to FW) + `ResponseQ` (read from FW). Each IOCB type as distinct struct.
- **Target mode (`qla_target.c`) as state machine.** Per-port `Port<S>` with explicit states; transitions consume self.
- **NVMe-FC integration** via typed nvme_fc API.
- **EDIF (encryption) typed.** Sessions, keys, parameters all typed with `kfree_sensitive` semantics for key material.
- **NPIV vports** typed as separate `Vport` instances; per-vport WWPN/WWNN allocator strict.
- **BSG passthrough** — every BSG IOCB has its input/output buffer sizes validated against the opcode-specific schema.

Grsec is mandatory. qla2xxx historically had multiple CVEs (CVE-2024-26603-class in BSG, several mailbox-handling bugs). FC drivers are particularly sensitive because they trust the FC fabric: a misbehaving FC switch or hostile node on the SAN could send malformed RSCN events / ELS frames.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct qla_hw_data` | HBA-wide HW state | `Qla<Phase>` |
| `struct scsi_qla_host` | per-vport state (one per WWPN, multi for NPIV) | `Vport` |
| `struct fc_port` | per-target FC port | `FcPort<State>` |
| `struct req_que` / `rsp_que` | per-CPU IOCB queues | `RequestQ` / `ResponseQ` |
| `struct mbx_cmd_32` | mailbox command | `MbxCmd<Op>` |
| `qla2x00_probe_one(pdev, ent)` / `_remove_one(pdev)` | PCI probe/remove | `Qla::probe` / `Drop` |
| `qla2x00_initialize_adapter(vha)` / `_chip_diag(vha)` | adapter init | `Qla::initialize` |
| `qla2x00_loop_resync(vha)` / `_topology_resolved(vha)` | loop / topology init | `Qla::resolve_topology` |
| `qla24xx_login_fabric(vha, fcport, ...)` / `_login_loop(...)` / `_logout_fabric(...)` | port login/logout | `FcPort::login` / `logout` |
| `qla2x00_fabric_resolve_target(vha)` / `_scan_target(...)` | discovery via Name Server | `Discovery::scan_targets` |
| `qla2x00_async_event(vha, rsp, mb)` | async event dispatch | `Qla::on_async_event` |
| `qla2x00_mailbox_command(vha, mcp)` / `_mailbox_command_vendor(...)` | mbx send | `MbxCmd::send` |
| `qla24xx_iocb(vha, rsp, mb, iocb)` | IOCB build + send | `Iocb::send` |
| `qla24xx_msix_*` / `qla2x00_isr(...)` | IRQ handlers | `Irq::*` |
| `qla24xx_start_scsi(scsi_cmd)` | SCSI cmd issue | `Qla::start_scsi` |
| `qla24xx_status_entry(...)` / `_status_cont_entry(...)` | SCSI completion | `Qla::on_status` |
| `qla24xx_abort_command(...)` / `_qla24xx_lun_reset(...)` / `_target_reset(...)` / `_host_reset(...)` | error recovery | `ErrorRecovery::*` |
| `qlt_target_init` / `qlt_unreg_sess(...)` / `_handle_command_event(...)` | target mode | `Target::*` |
| `qla_nvme_register_hba(vha)` / `_unregister_hba(...)` / `_post_cmd(...)` | NVMe-FC integration | `NvmeFc::*` |
| `qla24xx_edif_*` | EDIF (FC-SP-2) | `Edif::*` |
| `qla24xx_create_vp_index(...)` / `_delete_vp(...)` | NPIV vport | `Vport::create` / `delete` |
| `qla2x00_attach_target_vport(...)` | target attach | `Vport::attach_target` |
| `qla2xxx_bsg_request(...)` / `_bsg_timeout(...)` | BSG passthrough | `Bsg::request` |
| `qla2x00_get_data_rate(vha)` / `_set_data_rate(...)` | speed (8/16/32/64/128 Gbps) | `Speed::get` / `set` |
| `qla2x00_set_min_max_link_speed(...)` | speed limit | `Speed::set_range` |
| `qla2x00_get_flash_image(...)` / `_update_flash_image(...)` | FW image | `Flash::get` / `update` |
| `qla24xx_protect_ecc_*` | ECC + integrity check | `Ecc::*` |
| `qla24xx_fw_dump(...)` | FW core dump | `FwDump::capture` |
| `qla2x00_sysfs_*` | sysfs attrs | `Sysfs::*` |

## Compatibility contract

REQ-1: PCI ID table — every QLE25xx/26xx/27xx/28xx/2870 silicon + legacy QLA21xx/22xx/23xx (8/16/32/64/128 Gbps + older 2/4 Gbps for compat). Frozen.

REQ-2: FW image binding — per-chip FW required: `ql2xxx_fw.bin`, `ql2400_fw.bin`, `ql2500_fw.bin`, `ql8044_fw.bin`. Loaded at init via `request_firmware`; PSP-equivalent signature: QLogic-signed. Loaded into FW SRAM by HBA on `LOAD_FIRMWARE` mbx.

REQ-3: Mailbox protocol — 32 mailbox registers + opcode encoded in MBX0; in/out args in MBX1..N. Hundreds of opcodes covering all HBA functionality. Single mbx-at-a-time per host; serialization mandatory.

REQ-4: IOCB request queue — per-CPU request queues; SCSI cmds + management cmds posted as IOCBs. IOCB sizes vary by type (32B for short, 64B for full IO with sg-list).

REQ-5: IOCB response queue — per-CPU response queues; FW posts completion entries with status + sg-buf addresses.

REQ-6: FC topology — Point-to-point / FC-AL (loop) / FL_Port (fabric) — discovered via FLOGI / PLOGI flow. RSCN updates handled async.

REQ-7: Port login flow — PLOGI → PRLI (per-protocol login: FCP for SCSI, NVMe for NVMe-FC) → port available. State machine survives transient FC fabric disruptions.

REQ-8: Target mode — qla_target.c provides callbacks for LIO target stack. Per-LUN command flow tracked.

REQ-9: NVMe-FC — when target advertises NVMe-FC support via PRLI, nvme_fc transport bound; NVMe controller created.

REQ-10: NPIV — multiple WWPNs per physical port; each is a `scsi_qla_host`. Configured via sysfs.

REQ-11: EDIF — FC-SP-2 sessions: DH-CHAP auth + AES-GCM encrypt. Keys via NL extension; lifecycle managed.

REQ-12: BSG — passthrough for vendor tools (SAN management). Frozen ioctl set per `QL_VND_*`.

REQ-13: Error recovery — abort → LUN reset → target reset → host reset (full HBA reset) → adapter offline. Per-LUN timeout + escalation.

REQ-14: FW core dump — on FW exception, snapshot HBA state to host RAM for vendor analysis.

REQ-15: PCIe AER — pci_error_handlers; FW recovery via reset on AER event.

## Acceptance Criteria

- [ ] AC-1: `lspci` shows QLE2670 (16G); Rookery probes; FW loads; `dmesg | grep qla` shows expected init lines.
- [ ] AC-2: SAN-attached LUNs visible: `lsscsi` lists 100+ LUNs across multipath; reads/writes work.
- [ ] AC-3: NVMe-FC: SAN-attached NVMe namespace appears as `nvme0n1`; `nvme list` reports it; reads/writes at 32G line rate.
- [ ] AC-4: NPIV: create 4 vports via sysfs; each gets WWPN; each enumerates LUNs independently.
- [ ] AC-5: Fabric event: pull fiber → RSCN; multipath fails over; replug → RSCN; multipath fails back.
- [ ] AC-6: Target mode + LIO: present a LUN via LIO target on top of qla2xxx; remote initiator sees the LUN; reads/writes work.
- [ ] AC-7: EDIF: with peer supporting FC-SP-2, set up encrypted session via auth-cli; subsequent traffic AES-GCM encrypted (verify via FC analyzer).
- [ ] AC-8: 32 Gbps link: QLE2740 saturates at ~3.2 GB/s sustained read+write via fio.
- [ ] AC-9: FW reset: trigger HBA reset; recovery completes; multipath survives.
- [ ] AC-10: AER injection: clean recovery.

## Architecture

**Mailbox + IOCB split.** qla2xxx has two distinct host-to-HBA paths:
- **Mailbox** (MBX): synchronous, slow, used for setup + management. Driver writes mbx regs, FW reads + acts; response in mbx regs. One-at-a-time per host.
- **IOCB**: asynchronous, fast, used for IO + bulk operations. Driver writes IOCB into request queue (in host RAM, DMA-mapped); FW DMAs in + executes; posts completion IOCB into response queue + raises interrupt.

Mailbox: hundreds of opcodes for every HBA function. Hand-written per-opcode formatters (one of the bug-rich areas).
IOCB: ~30 types (CMD_TYPE_7 for SCSI, CMD_TYPE_NVME for NVMe, etc.).

**Discovery state machine.** On link up:
1. FLOGI (Fabric Login) to FC switch.
2. PLOGI to Name Server.
3. NS_GFPN_ID → list of fabric ports.
4. PLOGI to each port.
5. PRLI per port (negotiates SCSI vs NVMe).
6. Process Login complete → port usable.

Each port goes through `FcPort<S>` state machine: `Discovered → PlogiInProgress → PrliInProgress → Online → Offline → Recovery`.

**Async events.** RSCN (Registered State Change Notification) tells driver a fabric topology change happened. Driver scans NS + re-validates ports. PLOGI/PRLI may be redone. Edge cases: LUN added, LUN removed, fabric reconfigure.

**Target mode** (`qla_target.c`). When target mode enabled, qla2xxx receives FC frames from initiators and dispatches them to the LIO target stack. Per-initiator session tracking, per-LUN command flow, ATIO (Accept Target IO) → CTIO (Continue Target IO) → completion. Long history of bugs here.

**NVMe-FC integration.** When PRLI advertises NVMe support, `nvme_fc_register_localport` registers the local end + `nvme_fc_register_remoteport` registers the discovered NVMe target. nvme-core creates NVMe controllers + namespaces. Data path: NVMe commands → IOCB CMD_TYPE_NVME.

**EDIF (FC-SP-2).** Encryption: DH-CHAP authenticate (negotiate auth params), generate AES-GCM session key, all subsequent frames encrypted. Driver coordinates key install in FW (FW does the actual crypto), userspace tools (auth-cli) do the negotiation. Sessions auto-rekey periodically.

**NPIV.** Each `scsi_qla_host` (vport) has its own WWPN. Physical port runs FLOGI; each vport runs FDISC. Multi-tenant hosts use NPIV to give each tenant a distinct FC identity.

**Error recovery.** Per-LUN cmd timeout → abort. Abort failed → LUN reset. LUN reset failed → target reset. Target reset failed → host reset (full HBA bounce). Each step posts events to multipath (so upper layer can fail over).

**BSG passthrough.** Vendor tools (qaucli, scli) issue raw mailbox / IOCB commands via BSG. Driver validates the BSG IOCB payload header + size before forwarding.

## Hardening

- Mailbox cmd timeout (60s) enforced; past timeout, recovery triggered.
- IOCB request queue depth bounded.
- All async event payloads validated.
- BSG IOCB types validated against per-type expected size.
- Target mode session refcounts saturating.
- NVMe-FC controller lifetimes refcount-tracked.
- EDIF key material kfree_sensitive.
- NPIV vport count capped (silicon-dependent).

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — BSG ioctl ingress is the primary user surface; per-opcode size validation before copy_from_user. Sysfs attrs use bounded reads.
- **PAX_KERNEXEC** — scsi_host_template, IOCB handler tables, mbx-opcode dispatch, BSG dispatch table, async-event handler table, qlt target ops, nvme_fc ops, edif ops, `pci_error_handlers` all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — BSG ioctl, sysfs entry paths per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `qla_hw_data` refcount, per-vport refcount, per-fc_port refcount, per-target-session refcount, per-EDIF-session refcount, NVMe-FC controller refcount saturating.
- **PAX_MEMORY_SANITIZE** — IOCB DMA buffers cleared after submit (carry FCP CDB + data fragments). EDIF session keys cleared on session-end. Target-mode buffer pools sanitized at LUN unmap.
- **PAX_UDEREF** — SMAP/SMEP on BSG, sysfs, debugfs.
- **PAX_RAP / kCFI** — scsi_host_template ops, mbx-opcode dispatch, IOCB type dispatch, async-event dispatch, BSG handler table, qlt target ops all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/qla2xxx/*`) reveals IOCB queue addresses, mbx state, target session pointers; scrubbed for non-root.
- **GRKERNSEC_DMESG** — qla2xxx verbose logs (per-port discovery trace, RSCN details revealing fabric topology, NVMe controller creation) restricted from non-root.
- **CAP_SYS_RAWIO required on BSG** — BSG passthrough requires CAP_SYS_RAWIO. Historical CVE class (BSG ioctl missing capability check) closed.
- **BSG opcode allowlist + per-opcode size strict** — vendor cmds passed through BSG are validated against an allowlist; sizes checked per-opcode. Destructive opcodes (vendor FW reset, NVRAM write) require additional CAP_SYS_ADMIN.
- **FW image signature mandatory** — QLogic-signed FW images; unsigned refused. LOCKDOWN_INTEGRITY_MAX disables FW updates.
- **Mailbox cmd opcode allowlist per-vport** — vports (NPIV) restricted to a subset of mbx opcodes; admin opcodes (LIP, full reset) require physical-port issuance.
- **EDIF key handling** — encryption keys flow PCI-side only; kernel-side never holds plaintext key in long-lived state. Keys cleared from memory immediately after install into FW.
- **FC fabric trust boundary explicit** — fabric attackers can send hostile ELS frames + RSCN events + PLOGI requests. Driver validates: ELS frame parser bounded, port-WWPN-uniqueness enforced (peer cannot spoof a different WWPN), RSCN rate-limited (closes "fabric attacker triggers thousand-RSCN/sec DoS").
- **Target mode session per-initiator authentication** — when EDIF enabled, target accepts only authenticated initiators.
- **NPIV WWPN allocator deterministic + audited** — closes a path where dynamic-WWPN allocation could collide with a peer's WWPN.
- **FW core dump access gated** — coredump may contain in-flight IO data, encryption keys, LUN-mapped data; CAP_SYS_ADMIN required.
- **NVMe-FC controller per-WWPN scoped** — NVMe namespaces tagged with owning controller's WWPN; cross-WWPN namespace access refused.
- **Host reset rate-limit** — 3 in 60s, then HBA offline.
- **Aborted-cmd response handling strict** — abort response without a tracked SMID-equivalent dropped (closes UAF class).
- **Sysfs writes capability-checked** — every state-changing sysfs attr (port_param, fw_state_dump, edif key install) requires CAP_SYS_ADMIN.

Per-doc rationale: qla2xxx is the dominant FC HBA in enterprise SAN deployments. The FC fabric is a *trust boundary* — a misbehaving switch or hostile node on the SAN can send malformed RSCN events, ELS frames, or PLOGI requests. EDIF (FC-SP-2) provides crypto at the link layer for high-security environments. The BSG passthrough is the historical CVE source (CAP_SYS_RAWIO + opcode-allowlist closes it). The huge mailbox surface (hundreds of opcodes) + per-opcode hand-written formatters is the next-biggest source — typed mbx interface closes the format-bug class structurally. Target mode + NPIV double the attack surface; tenant isolation via WWPN + authenticated target sessions closes cross-tenant access. The FW image signature + LOCKDOWN_INTEGRITY_MAX gate closes the FW-substitution class.

## Open Questions

- [ ] Q1: qla_nx.c / qla_nx2.c — legacy iSCSI offload variants. Mostly deprecated; should Rookery support? Recommendation: minimal — covered by FCoE bridge code; punt iSCSI.
- [ ] Q2: 128 Gbps (QLE2870) is post-2024 silicon; check if all firmware paths preserved at the upstream baseline.
- [ ] Q3: EDIF + multi-pathing — encryption sessions are per-port, multi-pathing makes session establishment more complex. Need clear design for path-switch during active session.
- [ ] Q4: Target mode + NVMe-FC target — supported? Currently no nvmet-fc on qla2xxx; future work.
- [ ] Q5: SR-IOV — not supported in production qla2xxx; defer.

## Verification

- **Kani SAFETY**: prove FcPort state transitions are well-defined (no impossible-state-from-valid-state). Prove mailbox cmd issue serialization (no concurrent mbx).
- **TLA+**: model the discovery state machine with concurrent RSCN events + cable pulls. Check converges to consistent topology view; no stuck-in-pending state.
- **Verus**: functional spec of NS_GFPN_ID parser — for valid GS response, returns list of (port_id, wwpn); for malformed, returns -EINVAL.
- **Kani+Verus**: invariant that every active FcPort has a tracked PRLI state; every NVMe-FC controller maps to exactly one FcPort.
- **Integration**: enterprise SAN smoke — 24-LUN storage array attached via 32G FC; concurrent IO at line rate sustained 24h; NPIV with 16 vports; NVMe-FC bandwidth test; EDIF crypto throughput; multipath failover stress; cable-pull/reattach 1000 cycles.
- **Fuzz**: BSG ioctl fuzzer (payload mutations); RSCN payload fuzzer; ELS frame fuzzer.
- **Penetration**: unprivileged user with sysfs access attempts BSG passthrough — refused. CAP_SYS_RAWIO without CAP_SYS_ADMIN attempts FW reset — refused. Hostile peer attempts WWPN spoofing — refused.

## Out of Scope

- LIO target stack — separate Tier-3 (`drivers/target/`)
- FC transport class (`drivers/scsi/scsi_transport_fc.c`) — separate Tier-3 dependency
- NVMe-FC core (`drivers/nvme/host/fc.c` + `drivers/nvme/target/fc.c`) — separate Tier-3
- Emulex lpfc (sibling FC driver, ~150K lines) — separate Tier-3 (future iteration)
- SCSI mid-layer — `drivers/scsi/scsi_*.c` core Tier-3
- iSCSI offload (qla_nx.c, qla_nx2.c) — punted per Q1
