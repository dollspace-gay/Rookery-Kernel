# Tier-2: drivers/xen — Xen guest paravirt framework (xenbus + grant table + event channels + balloon + frontends)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/xen/
  - include/xen/
-->

## Summary

Tier-2 wrapper for Xen — paravirt framework for Linux running as Xen guest (PV / PVH / HVM with PV drivers). Components: **core** (`xenbus/` — XenBus IPC + watchpoint mechanism for xenstore; `events/` — Xen event channels (paravirt interrupts); `grant-table.c` — page-grant mechanism for cross-VM memory sharing; `gntdev.c` + `gntdev-common.c` + `gntdev-dmabuf.c` (`/dev/xen/gntdev`); `gntalloc.c`; `balloon.c` — memory balloon; `swiotlb-xen.c` — bounce buffer for non-coherent DMA in HVM; `evtchn.c` (`/dev/xen/evtchn`); `manage.c` — guest power-mgmt (suspend/resume/reboot); `mcelog.c` — MCE event upcall; `pcpu.c` — physical CPU presence under Xen; `pci.c` — PCI passthrough; `privcmd.c` (`/dev/xen/privcmd`); `xen-balloon.c`; `xen-front-pgdir-shbuf.c` — frontend page-dir shared buffer for virt graphics; `dbgp.c` — Xen debugger; `acpi.c` — Xen ACPI; `efi.c` — Xen EFI; `cpu_hotplug.c`; `mem-reservation.c`; `mmu.c`), **frontends** (`xen-pciback/` — Xen PCI backend; `xenbus_frontend.c`; `xen-front-pgdir-shbuf.c`; `xen-input.c` — paravirt input device frontend; `xen-acpi-cpufreq.c`; `xen-acpi-memhotplug.c`; `xen-acpi-pad.c`; `xen-acpi-processor.c`; `xen-front-pgdir-shbuf.c`; `xen-scsiback.c`; `xen-pcifront.c`).

PV-frontend block (`xen-blkfront.c`) lives under `drivers/block/`; PV-frontend net (`xen-netfront.c`/`xen-netback.c`) under `drivers/net/`; PV display (`xen-fbfront.c`) under `drivers/video/`.

## Compatibility contract — outline

- `/dev/xen/{evtchn, gntdev, gntalloc, hypercall, privcmd, xenbus, xenbus_backend}` chardev UAPIs byte-identical (libxc + libxenstore + xen-tools consume unchanged).
- xenstore protocol over xenbus byte-identical.
- Xen event channel + grant-table API source-compat for in-tree frontend drivers.
- `/sys/bus/xen-backend/devices/` + `/sys/bus/xen-frontend/devices/` byte-identical.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/xen/xenbus.md` | `xenbus/`: XenBus + xenstore client |
| `drivers/xen/events.md` | `events/`: Xen event channels |
| `drivers/xen/grant-table.md` | `grant-table.c` + `gntdev.c` + `gntdev-common.c` + `gntdev-dmabuf.c` + `gntalloc.c` |
| `drivers/xen/balloon.md` | `balloon.c` + `xen-balloon.c` + `mem-reservation.c` |
| `drivers/xen/swiotlb-xen.md` | `swiotlb-xen.c` |
| `drivers/xen/evtchn-userspace.md` | `evtchn.c`: `/dev/xen/evtchn` |
| `drivers/xen/privcmd.md` | `privcmd.c`: `/dev/xen/privcmd` |
| `drivers/xen/manage.md` | `manage.c`: power mgmt under Xen |
| `drivers/xen/pci.md` | `pci.c` + `xen-pciback/` + `xen-pcifront.c`: PCI passthrough |
| `drivers/xen/acpi.md` | `acpi.c` + `xen-acpi-*.c` |
| `drivers/xen/input-front.md` | `xen-input.c` |
| `drivers/xen/scsi-back.md` | `xen-scsiback.c` |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: `/dev/xen/*` chardev UAPI byte-identical.
- REQ-O2: xenstore protocol byte-identical.
- REQ-O3: TLA+ models (event-channel deliver + concurrent mask/unmask; grant-table refcount under cross-VM share).
- REQ-O4: AC: PV guest boots in upstream Xen hypervisor; PVH guest boots; xenstore-list root works.
- Hardening: row-1; `/dev/xen/privcmd` requires CAP_SYS_ADMIN; grant-table foreign-page mapping mediated.

## Out of Scope
Implementation code; ARM Xen; 32-bit-only paths.
