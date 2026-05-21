# Tier-3: drivers/scsi/lpfc/{lpfc_init,lpfc_sli,lpfc_els,lpfc_hbadisc,lpfc_mbox,lpfc_scsi,lpfc_nvme,lpfc_nvmet,lpfc_nportdisc,lpfc_ct,lpfc_attr,lpfc_bsg,lpfc_vport,lpfc_vmid,lpfc_mem,lpfc_debugfs}.c — Emulex / Broadcom Fibre Channel HBA (lpfc — every enterprise SAN, FC initiator + FC target + NVMe-FC)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/scsi/00-overview.md
sibling: drivers/scsi/qla2xxx.md (the other major FC HBA driver)
upstream-paths:
  - drivers/scsi/lpfc/lpfc_init.c                            (~16000 lines: HBA bring-up — PCI lifecycle, SLI mode select, FW load, port init)
  - drivers/scsi/lpfc/lpfc_sli.c                             (~22000 lines: SLI — Service Level Interface — request/response queue protocol)
  - drivers/scsi/lpfc/lpfc_els.c                             (~12580 lines: ELS — Extended Link Services — PLOGI/PRLI/RSCN/LOGO/FLOGI)
  - drivers/scsi/lpfc/lpfc_hbadisc.c                         (~7000 lines: HBA discovery state machine — node lifecycle, link events)
  - drivers/scsi/lpfc/lpfc_mbox.c                            (~2670 lines: mailbox cmd construction + dispatch)
  - drivers/scsi/lpfc/lpfc_scsi.c                            (~6000 lines: SCSI cmd path — IOCB build, completion)
  - drivers/scsi/lpfc/lpfc_nvme.c                            (~3000 lines: NVMe-FC initiator integration)
  - drivers/scsi/lpfc/lpfc_nvmet.c                           (~4000 lines: NVMe-FC target)
  - drivers/scsi/lpfc/lpfc_nportdisc.c                       (~3000 lines: N_Port discovery state machine per node)
  - drivers/scsi/lpfc/lpfc_ct.c                              (~3000 lines: FC Common Transport — Name Server + Management Server)
  - drivers/scsi/lpfc/lpfc_attr.c                            (~7000 lines: sysfs attrs, config knobs)
  - drivers/scsi/lpfc/lpfc_bsg.c                             (~4000 lines: BSG passthrough for OneCommand Manager + emlxs vendor tools)
  - drivers/scsi/lpfc/lpfc_vport.c                           (NPIV vport mgmt)
  - drivers/scsi/lpfc/lpfc_vmid.c                            (FC VMID — virtual machine identification for QoS)
  - drivers/scsi/lpfc/lpfc_mem.c                             (memory pool mgmt)
  - drivers/scsi/lpfc/lpfc_debugfs.c
  - drivers/scsi/lpfc/lpfc.h                                 (~3500 lines of types)
  - drivers/scsi/lpfc/lpfc_hw4.h                             (~20000 lines of FW ABI types — SLI-4)
  - drivers/scsi/lpfc/lpfc_hw.h                              (SLI-3 ABI types)
  - drivers/scsi/lpfc/lpfc_sli4.h
  - include/scsi/scsi_*.h
  - drivers/scsi/scsi_transport_fc.c                         (FC transport class)
-->

## Summary

lpfc is the Emulex (now Broadcom) Fibre Channel HBA driver — the other major enterprise FC HBA family in production, complementing qla2xxx. Covers every Emulex LightPulse silicon from the LP series through LPe series (LPe11000 8G, LPe12000 8/16G, LPe16000 16G, LPe31000 16G, LPe32000 32G, LPe35000 32/64G, LPe37000 64G, LPe38000 128G). Deployed in HP ProLiant, Dell PowerEdge, IBM, Lenovo, Cisco UCS, and Supermicro enterprise servers worldwide. Like qla2xxx, supports both initiator and target modes plus NVMe-FC.

The driver is the largest single-file driver in upstream Linux at ~103000 lines / 16 files — significantly larger than qla2xxx. The architectural complexity:

- **Two SLI versions** — SLI-3 (legacy, LP11000/12000 8G) and SLI-4 (modern, LPe16000+). The driver supports both, with SLI-version-conditional code paths throughout. SLI-4 has its own header (`lpfc_hw4.h`, 20K lines of FW ABI types).
- **FC + FCoE** — Emulex's LPe series silicon supports both native FC and Fibre-Channel-over-Ethernet. FCoE path adds ~10K lines of Ethernet-side glue.
- **NVMe-FC** — full initiator (lpfc_nvme.c) + target (lpfc_nvmet.c) support. Both are upstream.
- **NPIV** — N_Port ID Virtualization, per-port multi-WWPN.
- **VMID** — FC VMID for per-VM QoS tagging at the fabric layer (Cisco-style virtualization metric).

This Tier-3 covers all ~103000 lines (excluding the 20K-line ABI header which is auto-generated).

## Rust translation posture

lpfc shares many architectural patterns with qla2xxx — typed mbx interface, FC discovery FSM, NVMe-FC integration — but adds the SLI-3/SLI-4 dual-mode complexity. Translation:

- **`lpfc-core` crate** with SLI-version-conditional compilation: `Lpfc<Sli3>` vs `Lpfc<Sli4>` distinct types. Hardware methods only compile for the right version.
- **Typed SLI request/response queues.** SLI-4 has Work Queues (WQ) + Completion Queues (CQ) + Event Queues (EQ); SLI-3 had IO Control Blocks (IOCBs) in linear ring buffers. Typed `Wq<Type>`, `Cq<Type>`, `Eq` for SLI-4; `IocbRing` for SLI-3.
- **ELS frame builders typed.** Each ELS command (FLOGI, PLOGI, PRLI, LOGO, RSCN, ABTS) has a typed builder.
- **Node + discovery state machine.** `lpfc_nodelist` becomes typed `Node<NodeState>` with explicit transitions through DSM (Discovery State Machine) phases.
- **Mailbox interface typed** like qla2xxx — each opcode a method.
- **NVMe-FC target** — lpfc_nvmet.c is one of only a few upstream NVMe-FC target implementations. Closing UAF on session teardown is the headline grsec defense.
- **VMID tagging** — per-VM FC frame tagging for QoS; typed `VmId` allocator.

Grsec is mandatory: lpfc has had CVE history (CVE-2020-12652 inadequate authorization in BSG, multiple race CVEs in discovery and target paths). The huge code surface + SLI-version-dual-mode amplifies the audit difficulty.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct lpfc_hba` | HBA-wide HW state (SLI mode, FW caps, queues, etc.) | `Lpfc<Sli>` |
| `struct lpfc_vport` | per-vport state | `Vport` |
| `struct lpfc_nodelist` | per-N_Port node | `Node<State>` |
| `struct lpfc_iocbq` | IOCB queue entry | `Iocb` |
| `struct lpfc_sli_ring` | SLI-3 ring | `IocbRing` |
| `struct lpfc_queue` | SLI-4 WQ/CQ/EQ | `Q<Kind>` |
| `lpfc_pci_probe_one(pdev, ent)` / `_pci_remove_one(pdev)` | PCI probe/remove | `Lpfc::probe` / `Drop` |
| `lpfc_sli_brdrestart(phba)` / `lpfc_sli_brdready(phba)` | HBA reset | `Lpfc::reset` |
| `lpfc_sli_hba_setup(phba)` / `lpfc_sli4_hba_setup(phba)` | SLI-mode-specific bring-up | `Lpfc::setup_sli3` / `_sli4` |
| `lpfc_sli_post_sgl_pages(phba, ...)` | post sg-list pool | `SgPool::post` |
| `lpfc_sli_issue_mbox(phba, pmbox, flag)` | mbx issue | `Mbx::issue` |
| `lpfc_sli_issue_iocb(phba, ring_number, piocb, flag)` | IOCB issue | `Iocb::issue` |
| `lpfc_sli4_iocb_param_transfer(phba, ...)` | SLI-3 IOCB to SLI-4 WQE conv | `IocbConv::sli3_to_sli4` |
| `lpfc_disc_start(vport)` / `_state_machine(vport, ndlp, ...)` | discovery FSM | `Discovery::start` / `dsm` |
| `lpfc_els_flogi(vport, retry)` / `_plogi(...)` / `_prli(...)` / `_logo(...)` / `_rscn_recovery_check(...)` | ELS cmds | `Els::flogi` / etc. |
| `lpfc_nlp_set_state(vport, ndlp, state)` / `_nlp_remove(...)` | node state transitions | `Node::set_state` |
| `lpfc_queuecommand(host, scsi_cmd)` | SCSI cmd issue | `Lpfc::queue_cmd` |
| `lpfc_abort_handler(...)` / `_device_reset_handler(...)` / `_target_reset_handler(...)` / `_host_reset_handler(...)` | error recovery | `ErrorRecovery::*` |
| `lpfc_nvme_register_remoteport(...)` / `_unregister_remoteport(...)` | NVMe-FC remote register | `NvmeFc::register_remoteport` |
| `lpfc_nvme_init_post_io_cmd_iocb(...)` / `_nvme_io_cmd_iocb(...)` | NVMe IO submit | `NvmeFc::issue_io` |
| `lpfc_nvmet_setup_*` / `_xmt_ls_rsp(...)` / `_xmt_fcp_op(...)` | NVMe-FC target | `NvmeFcTarget::*` |
| `lpfc_ct_passthru(...)` / `_ct_handle_rcv(...)` | FC-CT | `Ct::passthru` / `_handle_rcv` |
| `lpfc_bsg_request(...)` / `_bsg_hba_resume(...)` | BSG | `Bsg::request` |
| `lpfc_create_vport_work(work)` / `lpfc_vport_create(...)` / `_vport_delete(...)` | NPIV | `Vport::create` / `delete` |
| `lpfc_vmid_init(phba)` / `_vmid_get_appid(...)` / `_vmid_destroy(...)` | VMID | `Vmid::*` |
| `lpfc_sli_handle_fast_ring_event(...)` / `_slow_ring_event(...)` / `_async_event_proc(...)` | IRQ event dispatch | `Irq::*` |
| `lpfc_fw_dump(phba)` / `_fw_diag_dump(...)` | FW core dump | `FwDump::*` |

## Compatibility contract

REQ-1: PCI ID table — every LP/LPe Emulex silicon: LP11000 (legacy), LPe11000/12000 (8G), LPe16000 (16G), LPe31000 (16G), LPe32000 (32G), LPe35000 (32/64G), LPe37000 (64G), LPe38000 (128G). FCoE-capable variants. Frozen.

REQ-2: FW image — per-silicon binary, loaded via `request_firmware`. Emulex-signed; signature checked by HBA hardware secure-boot.

REQ-3: SLI mode detection at probe — HBA reports SLI capability via register; driver paths SLI-3 or SLI-4 codepaths accordingly. Mixed-mode impossible.

REQ-4: SLI-4 queue model — Work Queues (WQ, driver → FW), Completion Queues (CQ, FW → driver), Event Queues (EQ, source of MSI-X interrupts; CQs are bound to EQs). Doorbell-driven.

REQ-5: SLI-3 ring model — single ring per channel, IOCB-based. Mostly compat path for older silicon.

REQ-6: ELS protocol — frozen against FC-LS-3 spec. PLOGI/PRLI/LOGO/RSCN/FLOGI/ABTS/PRLO/ADISC/PDISC supported.

REQ-7: Discovery state machine — per-node FSM: `UNUSED → PLOGI_ISSUE → PLOGI_COMPL → ADISC_ISSUE → ADISC_COMPL → REG_LOGIN_ISSUE → REG_LOGIN_COMPL → PRLI_ISSUE → PRLI_COMPL → UNMAPPED_NODE → MAPPED_NODE → PRLO_ISSUE → LOGO_ISSUE → NPR_NODE → ...`. Frozen state names.

REQ-8: NVMe-FC initiator — `lpfc_nvme.c` registers with nvme-fc transport; per-port NVMe controllers exposed.

REQ-9: NVMe-FC target — `lpfc_nvmet.c` registers with nvmet-fc; LUN export via NVMe-oFC.

REQ-10: NPIV — multi-vport per physical port.

REQ-11: VMID — per-VM FC frame tagging via FC4 VMID extension; QoS support on Cisco MDS fabrics.

REQ-12: FCoE — when CONFIG_LPFC_NL=y, Ethernet-side FCoE bridging supported. DCB integration for lossless FCoE.

REQ-13: Error recovery — same 4-level escalation as qla2xxx: abort → LUN reset → target reset → host reset.

REQ-14: BSG — OneCommand Manager + emlxs vendor tools issue passthrough mbx + ELS via BSG.

REQ-15: FW dump — on FW exception, register snapshot + queue state + last N IOCBs to host RAM.

## Acceptance Criteria

- [ ] AC-1: LPe32000 (32G) probes; SLI-4 mode detected; FW loaded; `dmesg | grep lpfc` matches upstream.
- [ ] AC-2: SAN-attached LUNs visible via lsscsi; reads/writes work.
- [ ] AC-3: NVMe-FC initiator: SAN-attached NVMe namespace as `nvme0n1`; line-rate IO.
- [ ] AC-4: NVMe-FC target: present a namespace via nvmet-fc on top of lpfc; remote initiator reads/writes.
- [ ] AC-5: NPIV 16 vports; each gets WWPN; concurrent IO independent.
- [ ] AC-6: VMID smoke: tag SCSI cmds with per-VM ID; fabric-side counters reflect tagging.
- [ ] AC-7: FCoE on LPe-FCoE silicon: FIP discovery completes; FCoE LUNs visible.
- [ ] AC-8: Cable pull: RSCN propagates; multipath fails over.
- [ ] AC-9: FW reset: HBA reset recovers cleanly.
- [ ] AC-10: 32G saturated link: fio sustains 3.2 GB/s read+write mix.

## Architecture

**SLI-3 vs SLI-4.** lpfc supports both. SLI-3 (legacy): single ring per channel for IOCBs. SLI-4 (modern): producer/consumer queues — WQ for submission, CQ for completion, EQ as IRQ source. SLI-4 supports many more queues (per-CPU) and higher concurrency. Driver detects mode at probe; subsequent code branches via `phba->sli_rev`. Rookery's `Lpfc<Sli3>` vs `Lpfc<Sli4>` typing closes the mode-confusion bug class (calling SLI-3 op on SLI-4 device = compile error).

**Discovery state machine.** Per-node (`lpfc_nodelist`) FSM. Each link event (port login, RSCN, cable pull) drives the FSM. Many states + many transitions; historical bug-rich area. Rookery's `Node<S>` typed transitions encode the FSM rules.

**ELS handling.** ELS commands flow through the discovery path. Each ELS frame has a typed builder + a typed parser for responses. Common ELS sequences (FLOGI → PLOGI → PRLI) chained via async completion callbacks.

**SCSI IO path.** scsi_cmd → IOCB build → enqueue to WQ/ring → FW executes → CQ entry → completion handler. IOCB carries sg-list (scatter-gather data buffer references). Per-LUN tag mgmt.

**NVMe-FC initiator.** Per discovered NVMe target, register with nvme-fc transport. NVMe commands flow through lpfc as IOCBs with CMD_TYPE_NVME flavor.

**NVMe-FC target.** When configured as target, exports namespaces via nvmet-fc. Per-initiator session tracking + per-namespace command flow.

**NPIV vports.** Multi-WWPN per physical port. Each vport runs its own discovery + has independent SCSI host.

**VMID.** Per-VM/per-container FC frame tagging. Frames carry an application identifier in a FC-4-extension field; fabric switches use the tag for per-tenant QoS.

**FCoE.** When silicon supports FCoE, lpfc bridges FC over Ethernet. FIP (FCoE Initialization Protocol) discovers the FCoE Forwarder; FC frames encapsulated in Ethernet.

**Error recovery.** Standard SCSI escalation: cmd abort → LUN reset → target reset → host reset.

## Hardening

- SLI-mode locked at probe; runtime change refused.
- Mailbox cmd timeout (60s); past timeout → recovery.
- Discovery FSM transitions validated; invalid transitions logged + recovery.
- ELS frame parser bounded.
- NVMe-FC target per-initiator session count capped.
- BSG ioctl per-opcode size validation.
- VMID allocator bounded.
- Host reset rate-limited.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — BSG ioctl ingress + sysfs attrs per-opcode size validation.
- **PAX_KERNEXEC** — `scsi_host_template`, ELS handler tables, IOCB type tables, async-event dispatch, BSG dispatch, NVMe-FC ops, discovery FSM transition tables, `pci_error_handlers` all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — BSG, sysfs, nvme-fc-passthrough entry paths per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `lpfc_hba` refcount, per-vport refcount, per-node refcount, per-session refcount, NVMe-FC controller refcount, BSG-job refcount saturating.
- **PAX_MEMORY_SANITIZE** — IOCB DMA buffers cleared after submit. NVMe-FC SQE/CQE buffers cleared. NVMe-FC target keys + per-namespace ID cleared on unmap.
- **PAX_UDEREF** — SMAP/SMEP on BSG, sysfs, debugfs.
- **PAX_RAP / kCFI** — scsi_host_template ops, ELS handlers, IOCB type dispatch, async-event handlers, BSG handlers, NVMe-FC ops, discovery FSM transitions all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/lpfc/*`) reveals queue addresses, ELS context pointers; scrubbed for non-root.
- **GRKERNSEC_DMESG** — lpfc verbose logs (discovery trace, ELS frame contents, RSCN details, NVMe target session events) restricted from non-root.
- **CAP_SYS_RAWIO required on BSG** — closes historical CVE-2020-12652 class (BSG missing authorization).
- **BSG opcode allowlist + per-opcode size strict** — destructive opcodes (FW reset, NVRAM write, FAW programming) require CAP_SYS_ADMIN.
- **FW image signature mandatory** — Emulex-signed FW; unsigned refused; LOCKDOWN_INTEGRITY_MAX disables FW updates.
- **Mailbox opcode allowlist per-vport** — NPIV vports restricted to vport-scope opcodes; admin-scope opcodes require physical-port issuance.
- **Fabric trust boundary explicit** — same as qla2xxx: hostile peer / hostile switch can send malformed RSCN / ELS / PLOGI. Defenses: ELS frame parser bounded, RSCN rate-limit, WWPN-uniqueness enforced, ADISC challenge cryptographically validated.
- **Discovery FSM transition allowlist** — `Node<S>` typed transitions; invalid transitions = compile error. Closes CVE class around discovery race conditions (CVE-2023-xxxx historical).
- **NVMe-FC target per-initiator authentication** — when fabric authentication enabled, only authenticated initiators allowed.
- **NPIV WWPN allocator deterministic+audited** — same as qla2xxx.
- **VMID allocator bounded** — per-vport max VMIDs; exhaustion returns -EBUSY rather than silently overwriting.
- **FW core dump access gated** — CAP_SYS_ADMIN required.
- **NVMe-FC namespace access per-controller scoped** — closes cross-tenant namespace access.
- **Aborted-cmd response handling strict** — refuse abort response without tracking ID; closes UAF class.
- **SLI mode pinned at probe** — runtime SLI-mode change refused (would be a structural bug + a privilege primitive).
- **VMID frame validation** — if a VMID-tagged frame arrives, the tag is validated against the vport's allocated set; foreign tags dropped.
- **Aerial-FW (Adapter Emulation in Emulation Layer) refused** — Emulex FW has a debug mode that emulates other vendor HBAs; refused in production.

Per-doc rationale: lpfc is the largest single-file driver in upstream Linux (~103K LOC). Architectural complexity is amplified by the SLI-3/SLI-4 dual-mode + FC + FCoE + NVMe-FC initiator + NVMe-FC target + NPIV + VMID feature stack. Historical CVEs concentrate in BSG passthrough (CVE-2020-12652) and discovery FSM races. The Rust translation's `Lpfc<Sli3>`/`Lpfc<Sli4>` typing + `Node<State>` typed transitions close the dominant bug classes structurally. The fabric-trust-boundary defenses parallel qla2xxx (ELS parser bounded, RSCN rate-limit, WWPN-uniqueness, ADISC challenge crypto). NVMe-FC target — one of the few upstream NVMe target backends — adds per-initiator authentication + cross-namespace access scoping.

## Open Questions

- [ ] Q1: SLI-3 maintenance — pre-LPe16000 silicon is mostly retired but some installed base remains. Recommendation: maintain via `Lpfc<Sli3>` typing; deprecation warning at probe.
- [ ] Q2: NVMe-FC target completeness — nvmet-fc + lpfc_nvmet is one of the few production NVMe target paths. Stress-test scale at upstream parity.
- [ ] Q3: FCoE deprecation — FCoE deployment is declining (most enterprise SAN moving to NVMe-FC directly). Keep FCoE support? Recommendation: keep (Cisco UCS deployments depend on it).
- [ ] Q4: VMID — Cisco fabric feature, somewhat niche. Defer if no scale demand? Recommendation: support, doc explicitly mentions fabric-dependency.

## Verification

- **Kani SAFETY**: prove SLI-mode-specific methods cannot be called on the wrong device. Prove Node<S> FSM transitions are well-defined.
- **TLA+**: model the discovery FSM with concurrent link events + RSCN bursts; verify converges to consistent state.
- **Verus**: functional spec of the ELS frame parser — for valid FC frame, decodes to typed ELS struct; for malformed, returns -EINVAL.
- **Kani+Verus**: invariant that every active Node has a tracked PRLI state; every NVMe-FC target session has matching initiator authentication state.
- **Integration**: enterprise SAN smoke (LPe32000 with 100 LUNs, NVMe-FC initiator + target, NPIV 32 vports, VMID tagging, FCoE on FCoE-capable silicon). Cable-pull stress. NVMe-FC target stress (high concurrent initiator count). FW reset injection.
- **Fuzz**: BSG ioctl fuzzer. ELS frame fuzzer. NVMe-FC SQE fuzzer.
- **Penetration**: BSG-without-CAP — refused. WWPN spoof — refused.

## Out of Scope

- LIO target stack — separate Tier-3
- NVMe-FC core — separate Tier-3
- FC transport class — separate Tier-3
- qla2xxx (sibling FC HBA) — covered in `qla2xxx.md`
- DCB (for FCoE QoS) — separate Tier-3
