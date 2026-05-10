# Tier-2: drivers/vhost — vhost framework (vhost-net + vhost-scsi + vhost-vsock + vhost-vdpa + vhost-user-side mux)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/vhost/
  - include/linux/vhost.h
  - include/uapi/linux/vhost.h
-->

## Summary

Tier-2 wrapper for vhost — kernel-side userspace-virtio backend that lets a host kernel module process a guest virtio queue without round-tripping to qemu/cloud-hypervisor user-process. Components: **vhost core** (`vhost.c` + `vhost.h`: framework — `vhost_dev` lifecycle + per-virtqueue access + KVM_IRQFD/KVM_IOEVENTFD wiring + per-vq worker kthread), **vhost-net** (`net.c`: kernel-side virtio-net backend; pairs with qemu virtio-net frontend; bypasses userspace for net I/O — every cloud VM uses this), **vhost-scsi** (`scsi.c`: kernel-side virtio-scsi backend; bridges to LIO target framework via TCM_VHOST), **vhost-vsock** (`vsock.c`: kernel-side virtio-vsock backend; pairs with AF_VSOCK socket family in `net/vmw_vsock/`), **vhost-vdpa** (`vdpa.c`: bridge to vDPA framework — userspace can drive a vDPA device via vhost-vdpa UAPI even if the device's vendor driver isn't a virtio driver per-se), **iotlb** (`iotlb.c`: per-vhost IOTLB for VIRTIO_F_IOMMU_PLATFORM guests), **vringh** (`vringh.c`: virtual virtqueue host-side interface helper), **vdpa-user** (`vdpa_user/`: VDPA-userspace bridge — vduse — let userspace implement a vDPA device backend).

## Compatibility contract — outline

- `/dev/vhost-{net,scsi,vsock,vdpa-N}` chardev IOCTLs byte-identical: `VHOST_GET_FEATURES`, `_SET_FEATURES`, `_SET_OWNER`, `_RESET_OWNER`, `_SET_MEM_TABLE`, `_SET_LOG_BASE`, `_SET_LOG_FD`, `_NET_SET_BACKEND`, `_SCSI_SET_ENDPOINT`, `_SCSI_CLEAR_ENDPOINT`, `_VSOCK_SET_GUEST_CID`, `_VSOCK_SET_RUNNING`, `_VDPA_GET_DEVICE_ID`, `_VDPA_GET_STATUS`, `_VDPA_SET_STATUS`, `_VDPA_GET_CONFIG`, `_VDPA_SET_CONFIG`, `_VDPA_SET_VRING_ENABLE`, `_VDPA_GET_VQS_COUNT`, `_VDPA_GET_GROUP_NUM`, `_VDPA_GET_AS_NUM`, `_VDPA_GET_VRING_GROUP`, `_VDPA_SET_GROUP_ASID`, `_SET_VRING_NUM`, `_SET_VRING_ADDR`, `_SET_VRING_BASE`, `_GET_VRING_BASE`, `_SET_VRING_KICK`, `_SET_VRING_CALL`, `_SET_VRING_ERR`, `_SET_VRING_BUSYLOOP_TIMEOUT`. Wire format byte-identical (qemu + cloud-hypervisor + dpdk vhost-user lib consume unchanged).
- vDPA bus on `/sys/bus/vdpa/` byte-identical (vdpa-cli consumes).

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/vhost/core.md` | `vhost.c` + `vhost.h`: framework |
| `drivers/vhost/iotlb.md` | `iotlb.c`: per-vhost IOTLB |
| `drivers/vhost/vringh.md` | `vringh.c`: vrh helper |
| `drivers/vhost/net.md` | `net.c`: vhost-net |
| `drivers/vhost/scsi.md` | `scsi.c`: vhost-scsi (via LIO TCM_VHOST) |
| `drivers/vhost/vsock.md` | `vsock.c`: vhost-vsock |
| `drivers/vhost/vdpa.md` | `vdpa.c`: vhost-vdpa bridge |
| `drivers/vhost/vduse.md` | `vdpa_user/`: vduse — userspace vDPA backend |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: `/dev/vhost-*` chardev UAPI byte-identical.
- REQ-O2: vhost-net + vhost-scsi + vhost-vsock + vhost-vdpa interop with respective qemu frontends.
- REQ-O3: vDPA bus + vdpa-cli consume unchanged.
- REQ-O4: TLA+ models (per-vq worker kthread + concurrent KVM_IRQFD/KVM_IOEVENTFD trigger; vhost-vdpa fd lifecycle vs vDPA driver remove).
- REQ-O5: AC: qemu vhost-net achieves line-rate vs userspace virtio; vsock guest↔host loopback works.
- Hardening: `/dev/vhost-*` open requires CAP_SYS_ADMIN (defends against userspace forging guest-visible virtio backend); per-VM iotlb-isolation enforced.

## Out of Scope
Implementation code; 32-bit-only paths.
