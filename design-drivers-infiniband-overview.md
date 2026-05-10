---
title: "Tier-2: drivers/infiniband — RDMA stack (verbs core + RDMA-CM + uverbs/uioctl + IB-MAD/CM + per-HCA hw + sw-rxe/siw + ULPs IPoIB/iSER/SRP/RTRS)"
tags: ["tier-2", "drivers", "rdma", "infiniband", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the InfiniBand / RDMA subsystem — the kernel framework that bridges userspace RDMA libraries (libibverbs, librdmacm, libfabric, UCX, MPI implementations OpenMPI/MPICH/Intel-MPI, NCCL, RCCL, GPUDirect-RDMA, NVMe-oF/RDMA, iSER, SMB-Direct) to RDMA-capable HCAs (HostChannelAdapter — Mellanox/NVIDIA ConnectX, Broadcom NetXtreme RoCE, AWS EFA, Chelsio T4/T5/T6 iWARP, Intel/idpf RDMA, AMD Pensando, Marvell, Huawei, Microsoft MANA-RDMA). Three concentric scopes:

1. **RDMA core** (`core/`): the framework — verbs vtable + uverbs+uioctl UAPI dispatch, MAD (Management Datagram) RPC subsystem, IB Subnet Manager Agent (SMA / SMI / SMP), Communication Manager (CM for IB, iWARP-CM for iWARP), RDMA-CM (rdma_cm — high-level connection manager wrapping IB-CM + iWARP-CM + RoCE under one API), counters / lag / restrack / nldev (rdmatool netlink), umem (userspace-memory pinning + ODP — On-Demand Paging), umem_dmabuf (dmabuf import for GPUDirect), CQ pool, FRMR pool, MR pool, multicast, addr (IP↔GID resolution), cma_configfs, cgroup (RDMA cgroup-v1+v2 controller for verbs object accounting)
2. **HW drivers** (`hw/`): per-HCA drivers — `mlx5/` (Mellanox/NVIDIA ConnectX-4/5/6/7/8 + BlueField — biggest single RDMA driver), `mlx4/` (legacy ConnectX-3), `bnxt_re/` + `bng_re/` (Broadcom RoCE), `efa/` (AWS EFA — SRD), `erdma/` (Alibaba E-RDMA), `hfi1/` (Intel Omni-Path 100Gb fabric — legacy), `hns/` (HiSilicon HNS RoCE), `ionic/` (AMD Pensando), `irdma/` (Intel iWARP/RoCE), `mana/` (Microsoft Azure MANA RDMA), `mthca/` (legacy Mellanox HCA — Tavor/Arbel/Sinai), `ocrdma/` (Emulex OneConnect RDMA), `qedr/` (QLogic/Marvell QED RDMA), `usnic/` (Cisco UCS VIC user-space NIC), `vmw_pvrdma/` (VMware paravirt RDMA), `cxgb4/` (Chelsio T4/T5/T6 iWARP)
3. **SW drivers** (`sw/`): software RDMA — `rxe/` (RDMA-over-Converged-Ethernet via software emulation, useful for testing without RoCE-capable NIC), `siw/` (Soft-iWARP — software-iWARP for testing), `rdmavt/` (Verbs-Transport — shared library for hfi1)
4. **ULPs** (`ulp/`): Upper-Layer Protocols — `ipoib/` (IP over InfiniBand — Ethernet-equivalent IP transport on IB fabric), `iser/` (iSCSI Extensions for RDMA — initiator), `isert/` (iSER target — pairs with LIO target), `srp/` (SCSI RDMA Protocol — initiator), `srpt/` (SRP target), `rtrs/` (RDMA Transport — block + scsi-like fast-path for storage)

Heavily cross-referenced from `drivers/net/ethernet/00-overview.md` (mlx5/bnxt/ice/idpf RDMA aux-bus split off from netdev), `drivers/iommu/00-overview.md` (PASID + ATS/PRI for SVA-RDMA), `drivers/nvme/00-overview.md` (NVMe-oF/RDMA host transport + nvmet-rdma target), `drivers/scsi/00-overview.md` (iSCSI/SRP transport classes), `drivers/gpu/drm/00-overview.md` (dma-buf import for GPUDirect-RDMA), `drivers/pci/00-overview.md` (PCIe binding + p2pdma + ATS), `drivers/vfio/00-overview.md` (VFIO-IB-passthrough), `drivers/dma-buf/` (cross-driver buffer import), `kernel/cgroup/00-overview.md` (RDMA cgroup controller).

### Out of Scope

- Per-Tier-3 (Phase D) — per-driver docs arrive incrementally
- Omni-Path (`hw/hfi1/`) — Intel EOL'd; keep present compile-gated
- mthca / mlx4 / ocrdma — legacy keep compile-gated
- 32-bit-only paths
- Implementation code

### components

### Core (`drivers/infiniband/core/`)

- **device** (`device.c`): `ib_device` lifecycle — register/unregister, per-device GID table mgmt, port state, async-event distribution
- **verbs** (`verbs.c`): generic verbs — `ib_alloc_pd`/`_dealloc_pd`, `ib_create_qp`/`_destroy_qp`, `ib_create_cq`/`_destroy_cq`, `ib_reg_mr`/`_dereg_mr`, `ib_post_send`/`_post_recv`, etc.
- **uverbs / uverbs_ioctl** (`uverbs_main.c` + `uverbs_cmd.c` + `uverbs_marshall.c` + `uverbs_std_types*.c` + `uverbs_uapi.c` + `uverbs_ioctl.c` + `rdma_core.c`): `/dev/infiniband/uverbsN` chardev — UAPI verbs dispatch (legacy write-cmd + new ioctl-based)
- **rdma_core** (`rdma_core.c`): per-context object table, refcount, file-fd lifecycle
- **ib_core_uverbs** (`ib_core_uverbs.c`): mmap-helper for verbs-mmap
- **MAD** (`mad.c` + `mad_priv.h` + `mad_rmpp.c` + `agent.c`): IB Management Datagram subsystem — SMP (Subnet Management Packet), GMP (General Management Packet), RMPP segmentation
- **CM** (`cm.c` + `cm_msgs.h` + `cm_trace.c`): IB Communication Manager — IB-spec connection establishment via REQ/REP/RTU
- **iwcm** (`iwcm.c` + `iwpm_msg.c` + `iwpm_util.c`): iWARP-Connection-Manager (separate from IB-CM since iWARP uses different conn establishment)
- **CMA** (`cma.c` + `cma_configfs.c` + `cma_priv.h` + `cma_trace.c`): RDMA-CM (rdma_cm) — high-level CM API wrapping IB-CM + iWARP-CM + RoCE behind unified `rdma_create_id` / `rdma_resolve_addr` / `rdma_resolve_route` / `rdma_connect` / `rdma_listen` / `rdma_accept` / `rdma_disconnect`
- **addr** (`addr.c`): IP↔GID resolution + ARP-equivalent
- **multicast** (`multicast.c`): IB multicast group join/leave
- **cache** (`cache.c`): GID + PKey cache per port
- **cq** (`cq.c`): CQ poll-context (workqueue-driven CQ event delivery + IRQ-affinity binding)
- **mr_pool** (`mr_pool.c`): pooled MR alloc for FRMR (Fast Registration Memory Region) reuse
- **frmr_pools** (`frmr_pools.c`): per-QP FRMR pool
- **counters** (`counters.c`): per-device + per-port counters (RxBytes, TxBytes, RxErrors, …)
- **netlink / nldev** (`netlink.c` + `nldev.c`): NETLINK_RDMA + nldev for `rdmatool` userspace
- **lag** (`lag.c`): RoCE LAG (link aggregation across RoCE ports for failover/bonding)
- **restrack** (`restrack.c`): per-resource tracking — every alloc'd PD/QP/CQ/MR with owner pid + comm; exposed via `rdmatool resource show`
- **packer** (`packer.c`): MAD packet packer/unpacker
- **ucma** (`ucma.c`): `/dev/infiniband/rdma_cm` chardev — RDMA-CM UAPI
- **umem** (`umem.c` + `umem_odp.c` + `umem_dmabuf.c`): userspace memory pinning + ODP (On-Demand Paging — page-fault into device pgtable via mmu_notifier) + dmabuf import (GPUDirect-RDMA)
- **iter** (`iter.c`): generic per-device iterator
- **opa_smi** (`opa_smi.h`): Omni-Path SMI helpers
- **cgroup** (`cgroup.c`): RDMA cgroup-v1+v2 controller — per-cgroup verbs-object accounting

### HW drivers (`drivers/infiniband/hw/`)

| Driver | Coverage |
|---|---|
| `mlx5/` | NVIDIA/Mellanox ConnectX-4/5/6/7/8 + BlueField DPU RDMA — IB + RoCE + RoCE v2 + EE-IB + DCT + ODP + SVA + GPU-direct |
| `mlx4/` | NVIDIA/Mellanox ConnectX-3 (legacy) |
| `bnxt_re/` + `bng_re/` | Broadcom NetXtreme-E RoCE |
| `efa/` | AWS Elastic Fabric Adapter (SRD — Scalable Reliable Datagram) |
| `erdma/` | Alibaba E-RDMA |
| `hfi1/` | Intel Omni-Path 100Gb (legacy / EOL) |
| `hns/` | HiSilicon HNS RoCE |
| `ionic/` | AMD Pensando RoCE |
| `irdma/` | Intel iWARP + RoCE (E810 + X722) |
| `mana/` | Microsoft Azure MANA RDMA |
| `mthca/` | Mellanox Tavor/Arbel/Sinai (legacy — keep compile-gated) |
| `ocrdma/` | Emulex OneConnect RDMA (legacy) |
| `qedr/` | QLogic/Marvell QED RDMA |
| `usnic/` | Cisco UCS VIC user-space NIC |
| `vmw_pvrdma/` | VMware paravirt RDMA |
| `cxgb4/` | Chelsio T4/T5/T6 iWARP |

### SW drivers (`drivers/infiniband/sw/`)

- `rxe/` — Soft-RoCE — pure software RDMA-over-Ethernet (no special HW required); useful for testing + validation
- `siw/` — Soft-iWARP — pure software iWARP
- `rdmavt/` — shared lib for hfi1 (Verbs-Transport)

### ULPs (`drivers/infiniband/ulp/`)

- **ipoib** — IP-over-InfiniBand (presents as a netdev `ib<N>`; CM-mode + datagram-mode + connected-mode)
- **iser** — iSCSI Extensions for RDMA — initiator (consumed by libiscsi)
- **isert** — iSER target — pairs with LIO target framework (`drivers/target/iscsi/iscsi_target_isert*.c` actually but cross-coupled)
- **srp** — SCSI RDMA Protocol initiator (legacy IB block-storage transport)
- **srpt** — SRP target — pairs with LIO target
- **rtrs** — RDMA Transport (RNBD — block-storage replacement for SRP/iSER on RDMA)

### scope

This Tier-2 governs:
- `/home/doll/linux-src/drivers/infiniband/` (~50+ subdirs covering thousands of files)
- Public headers `include/rdma/` (~30 headers)
- UAPI `include/uapi/rdma/` (~30 per-driver UAPIs)

PCI binding for the HW drivers cross-refs `drivers/pci/00-overview.md`. RDMA aux-bus integration for mlx5/bnxt/ice/idpf cross-refs `drivers/net/ethernet/00-overview.md`.

### compatibility contract — outline

### `/dev/infiniband/uverbs<N>` chardev

UAPI byte-identical so libibverbs + librdmacm + libfabric + libibumad + libibmad + perftest + libnl-route consume unchanged. Two dispatch paths:
- **Legacy write-cmd**: `write(uverbs_fd, struct ib_uverbs_cmd_hdr + payload)` — `IB_USER_VERBS_CMD_*` ioctl-codes-as-write-byte
- **New ioctl-based**: `ioctl(uverbs_fd, RDMA_VERBS_IOCTL, struct ib_uverbs_ioctl_hdr)` with object/method/attrs encoded in the header — extensible UAPI introduced ~5.0 baseline

Wire format byte-identical including alignment + reserved bytes.

### `/dev/infiniband/rdma_cm` chardev

UAPI for RDMA-CM (`librdmacm`). `RDMA_USER_CM_CMD_*` byte-identical.

### `/dev/infiniband/issm<N>` + `/dev/infiniband/umad<N>`

MAD chardevs for userspace SM (Subnet Manager) + general MAD agents (consumed by opensm + ibdiagnet + perfquery + ibping).

### sysfs surface

Per-device `/sys/class/infiniband/<device>/`:
- `node_desc`, `node_type`, `node_guid`, `sys_image_guid`, `fw_ver`, `hca_type`, `board_id`, `hw_rev`
- Per-port `ports/<N>/`: `state`, `phys_state`, `lid`, `lid_mask_count`, `gid_tbl_len`, `pkey_tbl_len`, `link_layer`, `rate`, `cap_mask`, `gids/<N>`, `pkeys/<N>`, `counters/{port_xmit_data, port_rcv_data, port_xmit_packets, port_rcv_packets, ...}`, `hw_counters/{...}`

Layout + content byte-identical so `ibv_devinfo`, `ibv_devices`, `ibstat`, `ibstatus`, `iblinkinfo`, `ibportstate`, `ibnetdiscover` consume unchanged.

### NETLINK_RDMA + rdmatool

Per-resource (PD, QP, CQ, MR, CTX, CM_ID, CTX_FB) introspection via `rdmatool resource show`. Wire format byte-identical so `rdmatool` from iproute2 works.

### umem ODP / dmabuf import

ODP (On-Demand Paging) — userspace registers a memory region with `IBV_ACCESS_ON_DEMAND`; pages are pinned lazily via mmu_notifier when the device faults. Identical to upstream behavior.

dmabuf import — `IBV_REG_DMABUF_MR` (newer UAPI) registers an external dmabuf as MR for GPUDirect-RDMA. UAPI byte-identical for nccl + RCCL + GPU-direct workflows.

### RDMA cgroup controller

cgroup-v1 + cgroup-v2 controller `rdma`/`io.rdma`: per-cgroup verbs-object limits (handle / count). Cross-ref `kernel/cgroup/rdma.md` future Tier-3.

### Tracepoints

`events/{ib_mad, ib_cm, iw_cm, rdma_cma, ib_iser, isert, srpt}/` byte-identical names + format.

### Module params

Per-HW-driver module params — frozen names so distro `/etc/modprobe.d/mlx5.conf` etc. apply identically. Notable: `mlx5_core.prof_sel`, `mlx5_core.guids`, `mlx5_core.node_guid`, `mlx5_core.debug_mask`, `mlx5_core.probe_vf`, `mlx5_core.num_of_groups`, `mlx5_core.lro_timeout_period_usecs`.

### iSER / SRP discovery

iSER initiator discovers targets via iSCSI iSNS or sysfs-add; same as iSCSI/TCP. SRP initiator discovers via `ibsrpdm`. Per-LUN block-device appears as `/dev/sd<X>`. Both work transparently with multipath-tools.

### IPoIB

`ib<N>` netdev presents as Ethernet-like for IP traffic. Connected-mode (CM, default for high MTU) + datagram-mode (default for low MTU). Behavior identical so DHCP + standard networking works.

### vfio-pci passthrough of RDMA HCAs

ConnectX-VF (SR-IOV) bound to vfio-pci enables guest to see HCA directly. Cross-ref `drivers/vfio/00-overview.md`. Behavior identical.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/infiniband/core-device.md` | `core/device.c` + `cache.c`: ib_device lifecycle + GID/PKey cache |
| `drivers/infiniband/core-verbs.md` | `core/verbs.c`: generic verbs (PD/QP/CQ/MR/SRQ/AH ops) |
| `drivers/infiniband/core-uverbs.md` | `core/uverbs_*.c` + `rdma_core.c` + `ib_core_uverbs.c`: `/dev/infiniband/uverbsN` UAPI |
| `drivers/infiniband/core-mad.md` | `core/mad*.c` + `agent.c` + `packer.c`: MAD subsystem |
| `drivers/infiniband/core-cm.md` | `core/cm*.c`: IB Communication Manager |
| `drivers/infiniband/core-iwcm.md` | `core/iwcm*.c` + `iwpm_*.c`: iWARP CM |
| `drivers/infiniband/core-cma.md` | `core/cma*.c` + `addr.c`: RDMA-CM (unified CM) |
| `drivers/infiniband/core-ucma.md` | `core/ucma.c`: `/dev/infiniband/rdma_cm` UAPI |
| `drivers/infiniband/core-multicast.md` | `core/multicast.c`: IB multicast |
| `drivers/infiniband/core-cq.md` | `core/cq.c`: CQ poll-context + IRQ binding |
| `drivers/infiniband/core-mr-pool.md` | `core/mr_pool.c` + `frmr_pools.c`: MR pool + FRMR pool |
| `drivers/infiniband/core-counters.md` | `core/counters.c`: per-port counters |
| `drivers/infiniband/core-restrack.md` | `core/restrack.c`: per-resource tracking |
| `drivers/infiniband/core-netlink.md` | `core/netlink.c` + `nldev.c`: NETLINK_RDMA + rdmatool |
| `drivers/infiniband/core-lag.md` | `core/lag.c`: RoCE LAG |
| `drivers/infiniband/core-umem.md` | `core/umem*.c`: userspace memory pinning + ODP + dmabuf import |
| `drivers/infiniband/core-cgroup.md` | `core/cgroup.c`: RDMA cgroup |
| `drivers/infiniband/hw-mlx5.md` | `hw/mlx5/`: ConnectX-4/5/6/7/8 + BlueField |
| `drivers/infiniband/hw-mlx4.md` | `hw/mlx4/`: ConnectX-3 (legacy) |
| `drivers/infiniband/hw-bnxt-re.md` | `hw/bnxt_re/` + `hw/bng_re/`: Broadcom RoCE |
| `drivers/infiniband/hw-efa.md` | `hw/efa/`: AWS EFA SRD |
| `drivers/infiniband/hw-erdma.md` | `hw/erdma/`: Alibaba E-RDMA |
| `drivers/infiniband/hw-hns.md` | `hw/hns/`: HiSilicon HNS RoCE |
| `drivers/infiniband/hw-ionic.md` | `hw/ionic/`: AMD Pensando |
| `drivers/infiniband/hw-irdma.md` | `hw/irdma/`: Intel iWARP/RoCE |
| `drivers/infiniband/hw-mana.md` | `hw/mana/`: Azure MANA RDMA |
| `drivers/infiniband/hw-cxgb4.md` | `hw/cxgb4/`: Chelsio T4/T5/T6 iWARP |
| `drivers/infiniband/hw-qedr.md` | `hw/qedr/`: QLogic/Marvell QED |
| `drivers/infiniband/hw-vmw-pvrdma.md` | `hw/vmw_pvrdma/`: VMware paravirt RDMA |
| `drivers/infiniband/hw-usnic.md` | `hw/usnic/`: Cisco UCS VIC |
| `drivers/infiniband/sw-rxe.md` | `sw/rxe/`: Soft-RoCE |
| `drivers/infiniband/sw-siw.md` | `sw/siw/`: Soft-iWARP |
| `drivers/infiniband/sw-rdmavt.md` | `sw/rdmavt/`: shared lib for hfi1 |
| `drivers/infiniband/ulp-ipoib.md` | `ulp/ipoib/`: IP-over-InfiniBand |
| `drivers/infiniband/ulp-iser.md` | `ulp/iser/`: iSER initiator |
| `drivers/infiniband/ulp-isert.md` | `ulp/isert/`: iSER target |
| `drivers/infiniband/ulp-srp.md` | `ulp/srp/`: SRP initiator |
| `drivers/infiniband/ulp-srpt.md` | `ulp/srpt/`: SRP target |
| `drivers/infiniband/ulp-rtrs.md` | `ulp/rtrs/`: RDMA Transport (RNBD) |

### compatibility outline (top-level)

- REQ-O1: `/dev/infiniband/uverbs<N>` legacy write-cmd + new ioctl-based UAPI byte-identical (libibverbs + librdmacm + libfabric + UCX + every MPI runtime consume unchanged).
- REQ-O2: `/dev/infiniband/rdma_cm` (ucma) UAPI byte-identical.
- REQ-O3: `/dev/infiniband/{issm,umad}<N>` MAD chardev UAPI byte-identical (opensm + ibdiagnet consume unchanged).
- REQ-O4: Per-device + per-port sysfs surface byte-identical (`ibv_devinfo`, `ibstat`, `ibportstate`, `iblinkinfo`, `ibnetdiscover` consume unchanged).
- REQ-O5: NETLINK_RDMA + nldev wire format byte-identical (`rdmatool resource show` consumes unchanged).
- REQ-O6: ODP + dmabuf-import UAPI byte-identical (GPUDirect-RDMA + nccl/RCCL workflows work unchanged).
- REQ-O7: RDMA cgroup controller (v1 + v2) per-resource limits enforced identically.
- REQ-O8: All in-tree v0 HW drivers enumerate + probe + register per-device UAPI byte-identically (mlx5/bnxt_re/efa/erdma/hns/ionic/irdma/mana/cxgb4/qedr/vmw_pvrdma/mlx4-legacy).
- REQ-O9: Soft-RoCE (rxe) + Soft-iWARP (siw) work for testing without RDMA HW.
- REQ-O10: ULPs (IPoIB + iSER + SRP + RTRS + isert + srpt) work end-to-end with reference initiators/targets.
- REQ-O11: NVMe-oF/RDMA host transport + nvmet-rdma target work via this Tier-2's verbs framework (cross-ref `drivers/nvme/00-overview.md`).
- REQ-O12: TLA+ models declared at this Tier-2 (verbs uobject lifetime + uverbs concurrent commands + RDMA-CM connection state machine + ODP page-fault + GID cache consistency + multicast group join/leave).
- REQ-O13: Verus/Creusot Layer-4 functional contracts on uverbs object refcount, MR length arithmetic, ODP page-table walk bounds.
- REQ-O14: Hardening: row-1 features applied per `00-security-principles.md`, with RDMA-specific reinforcement (umem CAP_IPC_LOCK respected; QP per-pkey isolation; ucma listen + connect mediated; XRC isolation between processes).

### acceptance criteria (top-level)

- [ ] AC-O1: `ibv_devinfo` + `ibstat` + `ibstatus` on a ConnectX-6 reference HCA produces output byte-identical to upstream baseline. (covers REQ-O1, REQ-O4)
- [ ] AC-O2: `ibping` between two RDMA-capable nodes succeeds; `perftest` (`ib_send_bw`, `ib_write_bw`, `ib_read_bw`, `ib_atomic_bw`) achieves rates within 5% of upstream baseline. (covers REQ-O1, REQ-O8)
- [ ] AC-O3: OpenMPI hello-world over OFED transport works between two ConnectX nodes. (covers REQ-O1, REQ-O2)
- [ ] AC-O4: `rdmatool resource show` enumerates active QP/CQ/MR/PD with correct owners. (covers REQ-O5)
- [ ] AC-O5: NVMe-oF/RDMA host connects to nvmet-rdma target; IO works at expected throughput. (covers REQ-O11)
- [ ] AC-O6: iSER initiator connects to LIO isert target; LUN appears as `/dev/sd<X>`; IO works. (covers REQ-O10)
- [ ] AC-O7: IPoIB test: `ifconfig ib0 up; ping <peer>` works between two IB nodes. (covers REQ-O10)
- [ ] AC-O8: Soft-RoCE test: `rdma link add rxe0 type rxe netdev eth0` followed by `ib_send_bw` between two software-RoCE peers. (covers REQ-O9)
- [ ] AC-O9: GPUDirect-RDMA test: nccl-tests `all_reduce_perf` between two GPU+ConnectX nodes works at expected bandwidth. (covers REQ-O6)
- [ ] AC-O10: RDMA cgroup test: cgroup with hca_handle limit 100 prevents 101st PD allocation. (covers REQ-O7)
- [ ] AC-O11: kselftest `tools/testing/selftests/rdma/` passes. (covers REQ-O12, REQ-O13)
- [ ] AC-O12: rdma-core test suite (`build/python/tests/run_tests.py`) passes against in-tree drivers. (covers REQ-O12, REQ-O13)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/rdma/uobject_lifetime.tla` | `drivers/infiniband/core-uverbs.md` (proves: per-context uobject table — concurrent uverbs commands creating/destroying objects + uverbs-fd close never leak object or double-free; refcount-cycle handled correctly) |
| `models/rdma/cma_state.tla` | `drivers/infiniband/core-cma.md` (proves: RDMA-CM connection state machine — IDLE → ADDR_QUERY → ADDR_RESOLVED → ROUTE_QUERY → ROUTE_RESOLVED → CONNECT → ESTABLISHED → DISCONNECTED → DESTROYING; concurrent disconnect + connect-complete never produce stuck-state) |
| `models/rdma/odp_pgfault.tla` | `drivers/infiniband/core-umem.md` (proves: ODP page-fault — device fault → mmu_notifier-aware host pgwalk → fault inserted into device pgtable; concurrent host munmap + device fault never produces stale device pgtable mapping or skipped invalidation) |
| `models/rdma/gid_cache.tla` | `drivers/infiniband/core-device.md` (proves: per-port GID cache + cache-update propagation; concurrent GID-add/remove + AH-create using GID-index never produces AH using freed-GID-entry) |
| `models/rdma/multicast_join.tla` | `drivers/infiniband/core-multicast.md` (proves: multicast group join/leave + per-port reference-count; final-leave triggers MAD detach; subsequent join recreates correctly) |
| `models/rdma/cgroup_resource.tla` | `drivers/infiniband/core-cgroup.md` (proves: RDMA cgroup-v2 hca_handle / hca_object limits enforced under concurrent verbs alloc/free; per-cgroup counter never goes negative) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/infiniband/core-uverbs.md` | uobject refcount: every CREATE returns object with refcount=1; DESTROY decrements; close-during-DESTROY safe |
| `drivers/infiniband/core-umem.md` | `ib_umem_get` post: returned umem covers exactly the requested user-virtual range; page-pin reference held until ib_umem_release |
| `drivers/infiniband/core-mad.md` | MAD packet length validated against header before deref; agent dispatch uses per-class table indexed by mad_hdr.mgmt_class within bounds |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-ib_device + per-pd + per-qp + per-cq + per-mr + per-uobject + per-cm_id + per-rdma_id refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-driver `ib_device_ops`, per-driver `uverbs_object_def`, per-class verbs vtables `static const` | § Mandatory |
| **SIZE_OVERFLOW** | MR length + WR sge_count + per-QP send/recv-queue depth + MAD-RMPP packet-count multiplications use checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-driver ib_device_ops + per-uobject method dispatch tables + MAD class-handler tables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed uobject state + freed MR + freed PD cleared (carries cross-process MR backing memory + crypto session keys for RDMA-encrypted transports) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | RDMA has no JIT — N/A | § Mandatory |

### Row-2 (LSM-stackable) features for RDMA

LSM hooks called: `security_ib_*` family — `_alloc_security`, `_free_security`, `_endport_manage_subnet`, `_pkey_access`. SELinux + AppArmor + future GR-RBAC stackable LSM each mediate. File-LSM hooks on `/dev/infiniband/*` open + ioctl + read/write.

GR-RBAC adds:
- Per-role disallow `/dev/infiniband/uverbs*` open (denies userspace verbs access).
- Per-role disallow `/dev/infiniband/rdma_cm` open (denies userspace CM).
- Per-role disallow `/dev/infiniband/{issm,umad}*` open (denies userspace MAD agent — opensm needs CAP_NET_RAW + this).
- Per-role disallow `IBV_ACCESS_REMOTE_*` MR creation (defends against remote-write/atomic exposure).
- Per-role audit of every QP create + every PD alloc (resource accounting + intrusion detect).

### RDMA-specific reinforcement

- **umem CAP_IPC_LOCK accounting**: registered MR pages count against RLIMIT_MEMLOCK + per-cgroup memlock limit (defense against unprivileged process pinning all of host RAM).
- **PKey enforcement**: per-port PKey table mediates which QPs can communicate; SELinux-IB hook validates PKey access per-process (defense against tenant on shared IB fabric snooping other tenant's PKey traffic).
- **XRC isolation**: XRC (eXtended Reliable Connection) shares receive-queues across processes; Rookery enforces per-process XRC domain isolation by default; cross-process XRC requires explicit security label + LSM grant.
- **ucma listen + connect mediation**: cross-namespace ucma listen rejected (defends against listening on init-userns from container).
- **ODP page-fault flood-protection**: per-device ODP fault rate-limited (default 100k/sec); over-rate triggers WARN + per-device throttle.
- **Restrack visibility**: `rdmatool resource show` filters to current pid_namespace by default (defends against host-namespace info leak from container).
- **GPUDirect dmabuf import IOMMU validation**: imported dmabuf must have IOMMU mapping consistent with importing HCA; defense against guest forging dmabuf to bypass IOMMU isolation.
- **MAD agent CAP_NET_RAW + CAP_NET_ADMIN**: managing the IB subnet (registering as SM, sending SMP packets) requires CAP_NET_RAW for issm + CAP_NET_ADMIN for umad agent. Audit every umad agent registration.
- **Soft-RoCE / Soft-iWARP CAP_NET_ADMIN**: creating an rxe/siw instance requires CAP_NET_ADMIN; defends against unprivileged user creating fake RDMA device.
- **Per-driver firmware-load signature verification**: mlx5/bnxt_re/etc. firmware images verified before load via the kernel firmware-load LSM hook (cross-ref `drivers/base/firmware-loader.md`).

