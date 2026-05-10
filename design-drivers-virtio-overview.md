---
title: "Tier-2: drivers/virtio — virtio core (transports + ring + balloon + mem + input + RTC + admin + dma-buf + vDPA bridge)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for virtio — the paravirtualized device framework underneath every cloud guest VM (qemu, cloud-hypervisor, firecracker, crosvm, vbox-on-Linux-host, Hyper-V Linux guests increasingly). Owns the **virtqueue** ring abstraction (split-ring legacy + packed-ring 1.1), per-transport probes (**virtio-pci** modern + legacy + admin-q for SR-IOV management; **virtio-mmio** for ARM/RISC-V SoC + cloud-hypervisor; **virtio_vdpa** bridge to vDPA framework for offloaded virtqueues), and per-device meta-drivers (balloon, mem hotplug, input, RTC, dma-buf cross-process import, vsock — but vsock lives under `net/vmw_vsock/`). Per-class virtio drivers (virtio-net under `drivers/net/`, virtio-blk under `drivers/block/`, virtio-scsi under `drivers/scsi/`, virtio-gpu under `drivers/gpu/drm/`, virtio-fs under `fs/`, virtio-console under `drivers/char/`, virtio-crypto under `drivers/crypto/`, virtio-iommu under `drivers/iommu/`, virtio-snd under `sound/virtio/`) are owned by their respective Tier-2s but consume the framework here.

Files: `virtio.c` (bus core + driver/device match + probe), `virtio_ring.c` (split + packed ring impl), `virtio_pci_{common,modern,legacy,modern_dev,legacy_dev,admin_legacy_io}.c` (PCIe transport — modern is virtio-1.1+, legacy is virtio-0.95, admin-legacy-io provides admin VQ for SR-IOV), `virtio_mmio.c` (memory-mapped transport), `virtio_balloon.c` (memory ballooning host↔guest), `virtio_mem.c` (memory hotplug + hot-unplug), `virtio_input.c` (paravirt input device — keyboards/mice/tablets in VM), `virtio_dma_buf.c` (cross-process dma-buf import), `virtio_rtc_*.c` (paravirt RTC + PTP), `virtio_vdpa.c` (bridge to vDPA framework — vhost-vdpa userspace consumer), `virtio_anchor.c` (anchor for sound), `virtio_debug.c` (debug helpers).

### Acceptance Criteria

- [ ] AC-O1: qemu boots Linux guest with virtio-net + virtio-blk + virtio-rng + virtio-balloon + virtio-vsock + virtio-gpu + virtio-input + virtio-console — all enumerate and work.
- [ ] AC-O2: cloud-hypervisor boots Linux guest with virtio-mmio devices.
- [ ] AC-O3: virtio-mem hot-add 4 GB to running guest; `cat /proc/meminfo` reflects.
- [ ] AC-O4: vhost-vdpa test: vDPA device presented as virtio-net to guest; throughput within 5% of host-managed.
- [ ] AC-O5: kselftest `tools/testing/selftests/drivers/virtio/` passes.

### Out of Scope

- vsock (lives in `net/vmw_vsock/` — separate Tier-3)
- Per-class virtio drivers (each owned by their respective subsystem Tier-2)
- 32-bit-only paths
- Implementation code

### compatibility contract — outline

- Guest UAPI: virtio-1.1 spec compliance (every legacy + modern + transitional device handled). Transport feature negotiation via `VIRTIO_F_VERSION_1` + `_NOTIFY_ON_EMPTY` + `_RING_INDIRECT_DESC` + `_RING_PACKED` + `_RING_RESET` + `_IN_ORDER` + `_ORDER_PLATFORM` + `_SR_IOV` + `_NOTIFICATION_DATA` + `_NOTIF_CONFIG_DATA` + `_RING_RESET` byte-identical.
- PCI ID range `0x1000-0x107F` (legacy) + `0x1040-0x107F` (modern, ID = 0x1040 + virtio_device_id).
- MMIO transport magic + version + device-id register layout per virtio spec § 4.2.
- Per-device modalias `virtio:d<device_id>v<vendor_id>` byte-identical so depmod+modprobe load right driver.
- Admin VQ (CONFIG_VIRTIO_PCI_ADMIN_LEGACY) used by qemu for SR-IOV PF managing VFs.
- vDPA framework integration: `virtio_vdpa.c` presents vDPA devices as virtio devices on `vdpa` bus (cross-ref `drivers/vhost/`).

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/virtio/core.md` | `virtio.c` + `virtio_anchor.c`: bus core + match + probe + feature negotiation |
| `drivers/virtio/ring.md` | `virtio_ring.c`: split + packed ring impl; descriptor table; avail/used (split) or desc-state (packed) management; in-order optimization; indirect descriptors |
| `drivers/virtio/pci-modern.md` | `virtio_pci_modern.c` + `virtio_pci_modern_dev.c` + `virtio_pci_common.c`: virtio-PCI modern (1.1) transport |
| `drivers/virtio/pci-legacy.md` | `virtio_pci_legacy.c` + `virtio_pci_legacy_dev.c` + `virtio_pci_admin_legacy_io.c`: virtio-PCI legacy (0.95) + admin-VQ for SR-IOV |
| `drivers/virtio/mmio.md` | `virtio_mmio.c`: MMIO transport (ARM/RISC-V SoC, cloud-hypervisor) |
| `drivers/virtio/balloon.md` | `virtio_balloon.c`: memory ballooning |
| `drivers/virtio/mem.md` | `virtio_mem.c`: memory hot-add/remove |
| `drivers/virtio/input.md` | `virtio_input.c`: paravirt input device |
| `drivers/virtio/rtc.md` | `virtio_rtc_{class,driver,ptp,arm,internal}.c`: paravirt RTC + PTP source |
| `drivers/virtio/dma-buf.md` | `virtio_dma_buf.c`: cross-process dma-buf import |
| `drivers/virtio/vdpa-bridge.md` | `virtio_vdpa.c`: bridge to vDPA framework |
| `drivers/virtio/debug.md` | `virtio_debug.c` |

### compatibility outline

- REQ-O1: virtio-1.1 spec compliance (every guest in qemu / cloud-hypervisor / firecracker / crosvm / vbox runs unmodified).
- REQ-O2: Both split-ring + packed-ring transports work for every per-class consumer.
- REQ-O3: Modern + legacy + transitional virtio-PCI all probe correctly.
- REQ-O4: virtio-mmio works for cloud-hypervisor + firecracker.
- REQ-O5: Admin VQ supports qemu's SR-IOV PF↔VF management.
- REQ-O6: vDPA bridge presents vDPA devices as virtio devices to per-class drivers.
- REQ-O7: TLA+ models (virtqueue avail/used acq-rel ordering, packed-ring desc-state machine, balloon report-vq vs page-zone race).
- REQ-O8: Hardening: virtio devices treat host as untrusted (per-spec §6) — every config-space read sanity-checked.

### verification

| TLA+ Model | Owner |
|---|---|
| `models/virtio/split_ring.tla` | `drivers/virtio/ring.md` (proves: split-ring producer-consumer with avail-idx + used-idx acq-rel; concurrent kick + interrupt never miss notification) |
| `models/virtio/packed_ring.tla` | `drivers/virtio/ring.md` (proves: packed-ring descriptor avail-bit + used-bit + wrap-counter; in-order completion preserved) |
| `models/virtio/balloon_ranges.tla` | `drivers/virtio/balloon.md` (proves: balloon inflate/deflate + page-isolation handshake never produces double-freed page or double-allocated PFN to host) |

### hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-virtio_device + per-virtqueue refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-driver `virtio_driver`, per-transport `virtio_config_ops` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | descriptor-len + per-vq num arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed virtio device priv data cleared | § Default-on configurable |

Virtio-specific reinforcement: **host-untrusted per virtio-1.1 §6** — every config-space read of guest-visible struct (queue size, feature bits, status) is range-checked against spec maximums; descriptor-table head index validated against `vq->num` before deref; used-ring entry length validated against original chain length (defense against malicious host driving guest into UAF / OOB-write).

