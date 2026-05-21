# Tier-3: drivers/infiniband/hw/mlx5/{main,mr,cq,qp,srq,gsi,mad,fs,devx,umr,odp,counters,cong,cmd,ah,dm,dmah,wr,std_types,doorbell,data_direct,restrack,ib_rep,qos,qpc,mem,umr,mlx5_ib}.c — Mellanox/NVIDIA ConnectX RoCE/InfiniBand verbs (mlx5_ib — every HPC cluster, every AI training cluster, every cloud-low-latency tier — NDR/HDR/EDR InfiniBand + 200/400G RoCE)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/infiniband/00-overview.md
depends-on: drivers/net/ethernet/mellanox/mlx5/mlx5-core.md
upstream-paths:
  - drivers/infiniband/hw/mlx5/main.c                           (~5500 lines: ib_device registration, ib-verbs core, multi-port, IPv4/IPv6 RoCE GID mgmt)
  - drivers/infiniband/hw/mlx5/qp.c                             (~5000 lines: QP — Queue Pair — create/modify/destroy via FW cmd)
  - drivers/infiniband/hw/mlx5/cq.c                             (~1525 lines: CQ — Completion Queue)
  - drivers/infiniband/hw/mlx5/srq.c                            (SRQ — Shared Receive Queue)
  - drivers/infiniband/hw/mlx5/mr.c                             (~3000 lines: MR — Memory Region, MTT building)
  - drivers/infiniband/hw/mlx5/umr.c                            (UMR — User Memory Region update)
  - drivers/infiniband/hw/mlx5/odp.c                            (~2000 lines: ODP — On-Demand Paging, demand-paged MRs)
  - drivers/infiniband/hw/mlx5/ah.c                             (AH — Address Handle)
  - drivers/infiniband/hw/mlx5/gsi.c                            (GSI — General Service Interface, queue type for SMI/GMP)
  - drivers/infiniband/hw/mlx5/mad.c                            (~1500 lines: MAD — Management Datagram, SMI/GMP traffic)
  - drivers/infiniband/hw/mlx5/fs.c                             (~3500 lines: Flow Steering — RAW ETH QPs, raw packet matching)
  - drivers/infiniband/hw/mlx5/devx.c                           (~3240 lines: devx — direct verbs, user-mode FW cmd issuance with token auth)
  - drivers/infiniband/hw/mlx5/dm.c                             (DM — Device Memory, on-NIC scratchpad)
  - drivers/infiniband/hw/mlx5/cong.c                           (congestion control — DCQCN + ZTR)
  - drivers/infiniband/hw/mlx5/counters.c                       (per-QP/per-port counters)
  - drivers/infiniband/hw/mlx5/cmd.c                            (HW-cmd wrapper)
  - drivers/infiniband/hw/mlx5/doorbell.c                       (doorbell page mgmt)
  - drivers/infiniband/hw/mlx5/data_direct.c                    (Data Direct Connector — pre-allocated qp+mr+cq for DOCA / SmartNIC)
  - drivers/infiniband/hw/mlx5/ib_rep.c                         (eSwitch IB representor for switchdev SR-IOV mode)
  - drivers/infiniband/hw/mlx5/qos.c                            (QP QoS — Service Class + bandwidth)
  - drivers/infiniband/hw/mlx5/qpc.c                            (QP context cache)
  - drivers/infiniband/hw/mlx5/mem.c                            (memory mgmt helpers)
  - drivers/infiniband/hw/mlx5/dmah.c                           (DMA-able heap)
  - drivers/infiniband/hw/mlx5/restrack.c                       (resource-tracking for rdmatool)
  - drivers/infiniband/hw/mlx5/wr.c                             (work-request builders)
  - drivers/infiniband/hw/mlx5/std_types.c                      (standard verbs type ops)
-->

## Summary

mlx5_ib is the Mellanox/NVIDIA ConnectX RDMA verbs driver — the implementation of `ib_device` (InfiniBand + RoCE + RDMA-over-Converged-Ethernet) on top of `mlx5_core` (covered in `mellanox/mlx5/mlx5-core.md`). Every HPC cluster (TOP500), every AI training cluster, every NVIDIA DGX/HGX system, every cloud-low-latency tier (AWS EFA-equivalent, Azure HC-series HPC, OCI HPC), every modern enterprise storage SAN going RoCE NVMe-oF — they all use mlx5_ib. ConnectX-7 silicon currently supports 400G NDR InfiniBand + 800G with future ConnectX-8.

mlx5_ib is the production-deployed RDMA driver in Linux — uverbs (user-verbs) clients via libibverbs (perftest, libfabric, MPI, NCCL, UCX), kernel-verbs clients (nvme-rdma, srp, iser, rds, rxe). It provides:

- **InfiniBand** — full IB stack including SMI (Subnet Manager), GSI, multicast, partition (P_Key), QoS (Service Level).
- **RoCE** — RDMA over Converged Ethernet (RoCEv1 + RoCEv2). Mostly RoCEv2 in current production (encapsulated in UDP).
- **GID management** — per-port GID table with RoCE v1 + v2 entries, IPv4 + IPv6 source addresses.
- **Standard verbs** — QP (RC/UC/UD/RAW), CQ, MR, AH, MW, PD, SRQ.
- **DEVX** — direct-verbs interface for advanced user-space (DOCA, libxdp-rdma, custom RDMA frameworks). Allows user-mode FW-cmd issuance with token authentication.
- **ODP** — On-Demand Paging for memory regions: register a large VA range without pinning; pages fault-in on first access via HMM-style mmu_notifier.
- **Flow Steering RAW ETH** — bind QPs to specific packet match criteria (used for accelerated DPDK-RDMA hybrid workloads, kernel-bypass packet capture).
- **Congestion control** — DCQCN (Data Center QCN), ZTR-RTT, per-flow rate adapt.
- **Multipath + LAG-aware** — single ib_device spans 2 PFs in LAG; QP+CQ resources span both physical ports.

This Tier-3 covers ~34400 lines / 29 .c files.

## Rust translation posture

mlx5_ib is the most CVE-dense single driver in upstream Linux (security-research focus on RDMA verbs has produced many issues over the years). The translation must close known classes structurally:

- **`mlx5-ib-core` crate** — ib_device registration + main verbs.
- **`mlx5-ib-qp` crate** — QP create/modify/destroy with phase typing.
- **`mlx5-ib-mr` crate** — MR + UMR + ODP with mmu-notifier safety.
- **`mlx5-ib-fs` crate** — flow steering RAW ETH.
- **`mlx5-ib-devx` crate** — direct verbs with token auth.
- **`Mlx5Ib<Phase>`** typed lifecycle.
- **`Qp<S: QpState>`** typed state machine. IB QP states: RESET → INIT → RTR (Ready to Receive) → RTS (Ready to Send) → SQ Draining → SQ Error → Error. Each transition consumes self.
- **`Mr<Status>`** typed: `Unregistered` / `Registered<Pinned>` / `Registered<OnDemand>` / `Invalidated`.
- **DEVX token auth typed.**

Grsec is mandatory and the single most important section in this whole document set. Historical CVEs: CVE-2024-26583 (mr UAF on user-space race), CVE-2023-52455 (qp create with bad lkey), CVE-2022-3239 (qp state-machine race), many more.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mlx5_ib_dev` | per-ib_device state | `Mlx5Ib<Phase>` |
| `struct mlx5_ib_qp` | QP | `Qp<S>` |
| `struct mlx5_ib_cq` | CQ | `Cq` |
| `struct mlx5_ib_mr` | MR | `Mr<Status>` |
| `struct mlx5_ib_ah` | AH | `Ah` |
| `struct mlx5_ib_srq` | SRQ | `Srq` |
| `struct mlx5_ib_pd` | PD | `Pd` |
| `mlx5_ib_add(dev, ...)` / `_remove(...)` | ib_device add/remove | `Mlx5Ib::add` / `Drop` |
| `mlx5_ib_stage_init_init(dev)` / etc. (many stages) | staged init | `Mlx5Ib::init_stages` |
| `mlx5_ib_create_qp(...)` / `_modify_qp(...)` / `_destroy_qp(...)` | QP verbs | `Qp::create` / `_modify` / `Drop` |
| `mlx5_ib_create_cq(...)` / `_destroy_cq(...)` / `_poll_cq(...)` | CQ verbs | `Cq::*` |
| `mlx5_ib_get_dma_mr(...)` / `_alloc_mr(...)` / `_dereg_mr(...)` | MR verbs | `Mr::*` |
| `mlx5_ib_reg_user_mr(...)` | user-MR register (pinned) | `Mr::reg_user` |
| `mlx5_ib_reg_user_mr_dmabuf(...)` | dmabuf-import MR | `Mr::reg_dmabuf` |
| `mlx5_ib_advise_mr(...)` | mr advise | `Mr::advise` |
| `mlx5_ib_umr_update_mtts(...)` / `_umr_revoke_mr(...)` | UMR update | `Umr::*` |
| `mlx5_ib_mr_init_mtt(...)` / `_mr_revoke_*` | MTT init/revoke | `Mtt::init` / `revoke` |
| `mlx5_ib_odp_init(...)` / `_odp_mkey_invalidation(...)` / `_odp_handle_page_fault(...)` | ODP integration | `Odp::*` |
| `mlx5_ib_create_srq(...)` / `_destroy_srq(...)` | SRQ verbs | `Srq::*` |
| `mlx5_ib_create_ah(...)` / `_destroy_ah(...)` | AH verbs | `Ah::*` |
| `mlx5_ib_alloc_ucontext(...)` / `_dealloc_ucontext(...)` | user ctx | `Ucontext::*` |
| `mlx5_ib_alloc_pd(...)` / `_dealloc_pd(...)` | PD verbs | `Pd::*` |
| `mlx5_ib_mmap(...)` | uverbs mmap | `Mlx5Ib::mmap` |
| `mlx5_ib_devx_create(...)` / `_devx_obj_create(...)` / `_devx_obj_destroy(...)` / `_devx_other(...)` | devx | `Devx::*` |
| `mlx5_ib_create_flow(...)` / `_destroy_flow(...)` | flow rule install | `Flow::*` |
| `mlx5_ib_get_native_port_mtu(...)` / `_query_port(...)` | port info | `Port::*` |
| `mlx5_ib_process_mad(...)` / `_set_mad_offload(...)` | MAD | `Mad::*` |
| `mlx5_ib_get_gids(...)` / `_set_gid(...)` | GID mgmt | `Gid::*` |
| `mlx5_ib_cong_set(...)` / `_query(...)` | congestion control | `Cong::*` |
| `mlx5_ib_alloc_counters(...)` / `_dealloc_counters(...)` | counters | `Counters::*` |
| `mlx5_ib_rep_attach(...)` / `_detach(...)` | eSwitch IB representor (switchdev) | `IbRep::*` |
| `mlx5_ib_data_direct_*` | DOCA / SmartNIC data-direct | `DataDirect::*` |

## Compatibility contract

REQ-1: ib_device caps — every uverbs-exposable capability bit from `ib_device_attr` matches what upstream advertises per ConnectX silicon generation.

REQ-2: QP types — RC (Reliable Connected), UC (Unreliable Connected), UD (Unreliable Datagram), RAW_PACKET, RAW_IPV6, XRC_INI/TGT (Extended RC), GSI, SMI.

REQ-3: QP state machine — IB spec-compliant transitions. RESET → INIT → RTR → RTS + error/draining/sqerr/sqd transitions. Frozen.

REQ-4: MR ops — reg_user_mr (pinned), reg_user_mr_dmabuf (dmabuf import), alloc_mr (kernel), dereg_mr.

REQ-5: ODP — on-demand paging via mmu_notifier; per-MR ODP enabled; page-fault handler in driver.

REQ-6: UMR — User Memory Region update: in-place MR update via doorbell-driven cmd. Used to remap MR pages without dereg+reg.

REQ-7: GID tab — per-port GID table; RoCE v1 (GID = MAC) + v2 (GID = IP) entries; per-net-namespace RoCE GID scope.

REQ-8: MAD — SMI (subnet management) + GMP (general management). Per IB spec.

REQ-9: Flow Steering RAW ETH — bind QP_RAW_PACKET to a flow-matcher (n-tuple); used for DPDK-RDMA hybrid + custom packet capture.

REQ-10: DEVX — direct verbs: user mode issues raw FW cmds via uverbs with per-user token authentication. Restricted opcode allowlist per token scope.

REQ-11: Congestion control — DCQCN (Data Center Quantized Congestion Notification), ZTR-RTT. Per-QP config.

REQ-12: GSI proxy — GSI traffic proxied through dedicated GSI QPs.

REQ-13: SRP / iSER / nvme-rdma / RDS / rxe — kernel-verbs consumers; all interfaces preserved.

REQ-14: LAG-aware — single ib_device spans 2 PFs when LAG active; QPs created on either physical port; CQ-from-LAG-master.

REQ-15: Multi-port — ConnectX-7 silicon: 2x 200G ports; one ib_device per port.

REQ-16: Resource tracking for rdmatool — `restrack.c` exposes per-resource info to userspace.

## Acceptance Criteria

- [ ] AC-1: ConnectX-6 probes via mlx5_core; mlx5_ib_add registers ib_device; ibv_devinfo shows expected caps.
- [ ] AC-2: rdma-cm ping between two hosts: works.
- [ ] AC-3: NVMe-oF RoCE: target presents NVMe namespace via nvmet-rdma; host initiator connects via nvme-rdma; line-rate IO.
- [ ] AC-4: GPUDirect RDMA: NVIDIA GPU → mlx5_ib via dmabuf-MR; cross-host GPU memory transfer at line rate.
- [ ] AC-5: ODP smoke: register a 1 TB virtual MR without pinning; access pages via QP; HMM page-faults; transfers succeed.
- [ ] AC-6: Flow Steering RAW: DPDK-RDMA app captures UDP traffic via RAW_PACKET QP bound to a flow-matcher.
- [ ] AC-7: DEVX: a privileged app issues raw FW cmd via DEVX; token-authenticated; cmd succeeds.
- [ ] AC-8: LAG-aware: dual-port ConnectX-6 in LAG; create QP; failover one cable; QP survives via LAG failover.
- [ ] AC-9: Congestion control: DCQCN-enabled QP under congestion; rate adapts as designed.
- [ ] AC-10: GSI: subnet manager (opensm) successfully manages a small IB fabric.
- [ ] AC-11: Resource leak test: open/close 1M QPs in sequence; no slab growth; ibstat shows clean state.

## Architecture

**Verbs implementation atop mlx5_core.** Every verb (create_qp, modify_qp, reg_user_mr, etc.) lowers to one-or-more `mlx5_cmd_exec()` calls into the mlx5_core firmware command interface (covered in `mlx5-core.md`). Driver maintains the user-facing handle + tracking metadata; FW holds the actual HW state.

**QP state machine.** IB QP states (RESET/INIT/RTR/RTS/SQE/SQD/ERR) transition via `IB_QP_STATE` modify. Each transition has IB-spec-mandated allowed previous-states. Driver validates each transition before issuing the MODIFY_QP cmd to FW. Rookery's `Qp<S>` typestate makes invalid transitions compile errors.

**MR + UMR + ODP.**
- Standard MR: pin all pages via `ib_umem_get` (or `_dmabuf_get`); program MTT (Memory Translation Table) into HW; return mkey.
- UMR (User Memory Region update): doorbell-driven cmd to update MR's MTT in-place. Saves dereg+reg overhead for streaming workloads.
- ODP (On-Demand Paging): MR registered without pinning. mmu_notifier-backed. On QP access to unmapped page, RNR (Receiver Not Ready) NAK → page fault → fault handler walks user mm → maps via HMM → resume.

**Flow Steering RAW ETH.** RAW_PACKET QP bound to flow-matchers via the flow-table primitives in `fs.c`. Used by DPDK-RDMA (libibverbs experimental queue + raw flow) for kernel-bypass packet processing with RDMA semantics.

**DEVX (direct verbs).** Allows user mode to issue raw FW cmds via uverbs. Critical for advanced workloads (DOCA SDK, custom RDMA frameworks). Token-authenticated: per-token opcode allowlist; un-allowlisted opcodes refused.

**Congestion control.** DCQCN: HW-implemented per-flow rate control responding to ECN marking. Driver configures per-QP CC profile.

**LAG-aware.** When mlx5_core's LAG is active (covered in `mlx5-lag.md`), ib_device represents the LAG aggregate. QPs allocated from PF0 can fail over to PF1 transparently.

**Multi-port.** Each ConnectX silicon port becomes its own ib_device (typically). Multi-port ib_device for some silicon variants.

## Hardening

- Every uverbs ioctl arg validated against the spec's struct shape.
- QP state-machine transitions validated.
- MR pin reference count strict.
- ODP page-fault handler bounded (no infinite loops on persistently un-mappable VAs).
- DEVX opcode allowlist strict.
- Flow Steering rule install bounded.
- Per-user-ctx resource caps enforced.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — every uverbs ioctl + DEVX cmd has typed arg structs; bounded copy_from_user per uverbs framework.
- **PAX_KERNEXEC** — ib_device ops, uverbs ioctl dispatch, QP state-machine validators, MR/CQ/SRQ/AH op tables, DEVX dispatch, flow steering dispatch, ODP handlers, MAD handlers all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — every uverbs ioctl per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — per-QP, per-CQ, per-MR, per-PD, per-AH, per-SRQ, per-user-ctx, per-flow-rule refcounts all saturating. Underflow = hard panic.
- **PAX_MEMORY_SANITIZE** — MR DMA contexts cleared on dereg. QP state buffers cleared on destroy. Flow-steering rule action buffers (contain crypto keys for IPsec offload) cleared.
- **PAX_UDEREF** — SMAP/SMEP on every uverbs entry.
- **PAX_RAP / kCFI** — ib_device ops, uverbs ioctl dispatch, QP state validators, ODP fault handler, DEVX cmd dispatch, MAD dispatcher all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs + restrack output scrubbed for non-root. uverbs leak-resource info gated.
- **GRKERNSEC_DMESG** — mlx5_ib verbose logs (per-QP-create trace, ODP fault details with VAs, flow-steering rule install) restricted from non-root.
- **CAP_NET_ADMIN required** — privileged verbs (CREATE_FLOW for system-level, MODIFY_CC for global CC config, etc.).
- **CAP_NET_RAW required** — RAW_PACKET QP creation (kernel-bypass packet I/O).
- **DEVX CAP_NET_ADMIN + per-token opcode allowlist** — DEVX direct-verbs cmd issue requires CAP_NET_ADMIN; per-token opcode allowlist further restricts. Closes CVE class around DEVX over-grant.
- **QP state-machine transitions strict** — Rust `Qp<S>` typestate makes invalid transitions compile errors. Closes CVE-2022-3239-class QP state-machine race historical.
- **MR mmu_notifier release strict** — for ODP MRs, mmu_notifier ordering enforced: notifier completion before MR dereg can proceed. Closes CVE-2024-26583-class MR UAF.
- **lkey/rkey allocator entropy** — local/remote keys allocated with high entropy (not monotonic). Closes 'predictable rkey enables cross-process RDMA read/write' class. CVE class historical (rkey-prediction attacks).
- **DEVX UCTX_TOKEN auth strict** — token validation on every DEVX cmd; user-token mismatch refused. Closes 'compromised process steals other process's DEVX token' class.
- **Per-user-ctx resource quotas strict** — max QPs/CQs/MRs/AHs/PDs per user-ctx enforced (per `IB_UVERBS_RES_*` resource caps). Closes resource-exhaustion DoS.
- **MR length + offset bounds** — length validated against actual mapped VMA; offset validated against MR length; sg-list overflow checked. Closes CVE-2023-52455-class.
- **ODP page-fault rate-limit** — per-QP ODP fault rate-limited to prevent fault-storm DoS.
- **Flow Steering rule install per-user scoped** — RAW ETH flow rules installed via uverbs scoped to the user-ctx's privilege; cross-user rule injection refused.
- **Congestion control config CAP_NET_ADMIN** — global CC profile change CAP_NET_ADMIN; per-QP CC config user-allowed.
- **MAD frame source-LID validation** — incoming MADs validated; closes spoofed SMI / GMP frame class.
- **LAG-aware QP migration atomic** — QP failover from PF0 to PF1 via LAG goes through an explicit handoff; no race with QP destroy. Closes UAF-during-failover class.
- **Multi-port tenant isolation** — multi-port ib_device: QPs/CQs/MRs scoped to their owning port; cross-port resource access refused unless explicitly authorized.
- **dmabuf-MR import strict** — imported dmabufs must have matching DMA attributes; cross-driver dmabuf attribute-mismatch refused. Closes 'GPU memory imported with wrong DMA mask → UAF' class.
- **DEVX `IB_USER_VERBS_EX_CMD_QUERY_DEVICE_EX` etc. — query opcode list filtered** — uverbs query path returns only the caps the user-ctx is allowed to see (e.g. doesn't expose RAW_PACKET cap to a non-NET_RAW user).
- **GID write CAP_NET_ADMIN** — GID table changes (sgid_index modify) require CAP_NET_ADMIN.
- **Resource tracker leak detection** — every resource has a tracker entry; at user-ctx destroy, all entries must be released (else hard-panic per refcount underflow).
- **mlx5_ib eSwitch IB Representor scoped** — switchdev-mode SR-IOV IB representors: per-rep ib_device strictly isolated.

Per-doc rationale: mlx5_ib is the production-critical RDMA driver for HPC + AI training + cloud-low-latency. The historical CVE concentration is among the highest in the kernel (RDMA verbs is a security-research focus area). Defenses: Rust `Qp<S>` typestate closes QP state-machine race class structurally (the largest historical category). MR mmu_notifier release ordering closes ODP UAF class. lkey/rkey high-entropy allocation closes rkey-prediction class. DEVX token + opcode allowlist closes the direct-verbs escalation class. Per-user-ctx resource quotas close DoS classes. The dmabuf-MR import strict-validation closes the GPU-RDMA UAF class (introduced with GPUDirect). LAG-aware QP migration atomic closes the new failover-race class. mlx5_ib is the single most security-critical driver in this enumeration.

## Open Questions

- [ ] Q1: DEVX opcode allowlist — what's the right per-token scope granularity? Recommendation: per-DEVX-uctx-token allowlist set at obj-create time.
- [ ] Q2: GPUDirect RDMA + dmabuf — interaction with amdgpu/nouveau/i915 dmabuf exporters. Cross-driver dmabuf attribute mismatch.
- [ ] Q3: ODP — bounded fault-storm protection: hard cap on per-QP fault rate? Best as a per-user-ctx config.
- [ ] Q4: LAG-aware QP migration — failover semantics with in-flight operations need careful design.
- [ ] Q5: NVMe-oF RoCE on shared NIC — multi-tenant SRIOV-VF NVMe-oF: per-VF QPs need rkey-domain isolation.

## Verification

- **Kani SAFETY**: prove `Qp<S>` transitions cannot reach disallowed combinations. Prove MR mmu_notifier release ordering safe.
- **TLA+**: model ODP page-fault concurrent with MR dereg; check no UAF.
- **Verus**: functional spec of MR length+offset validation — for any (mr, offset, length), either accepts (in-bounds) or returns -EINVAL.
- **Kani+Verus**: invariant that every active QP has tracked CQs + MRs; cleanup at user-ctx destroy removes all.
- **Integration**: HPC RDMA ping-pong perftest, NVMe-oF RoCE line-rate, GPUDirect RDMA between A100 + ConnectX-6, ODP stress (1M page faults), DEVX smoke with DOCA SDK, LAG failover stress, rdma-tool resource tracking.
- **Fuzz**: uverbs ioctl fuzzer (every uverbs op); DEVX cmd fuzzer; flow-steering rule fuzzer; ODP fault sequence fuzzer.
- **Penetration**: cross-process rkey-prediction attempt — must fail. DEVX token from another user-ctx — must refuse. RAW_PACKET QP without CAP_NET_RAW — refused.

## Out of Scope

- mlx5_core (the host driver below) — covered in `mlx5-core.md`
- mlx5_vdpa — separate Tier-3
- mlx5e netdev — covered in `mellanox-mlx5-en.md`
- mlx5 eSwitch — covered in `eswitch.md`
- mlx5 SF — covered in `mlx5-sf.md`
- mlx5 LAG — covered in `mlx5-lag.md`
- RDMA core (`drivers/infiniband/core/`) — separate Tier-3
- uverbs framework — `drivers/infiniband/core/uverbs_*.c` separate Tier-3
- libibverbs userspace — out of kernel scope
- nvme-rdma / iSER / SRP / RDS — separate Tier-3 each
- OpenSM (Subnet Manager) — userspace, out of scope
