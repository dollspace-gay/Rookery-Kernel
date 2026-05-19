# Tier-3: drivers/dma/idxd/* — Intel DSA / IAA Data Accelerator

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/dma/00-overview.md
upstream-paths:
  - drivers/dma/idxd/init.c
  - drivers/dma/idxd/device.c
  - drivers/dma/idxd/cdev.c
  - drivers/dma/idxd/dma.c
  - drivers/dma/idxd/submit.c
  - drivers/dma/idxd/irq.c
  - drivers/dma/idxd/sysfs.c
  - drivers/dma/idxd/bus.c
  - drivers/dma/idxd/perfmon.c
  - drivers/dma/idxd/registers.h
  - drivers/dma/idxd/idxd.h
  - include/uapi/linux/idxd.h
-->

## Summary

IDXD is the Linux driver family for Intel's data accelerators: **DSA** (Data Streaming Accelerator, Sapphire Rapids+ — memcpy / memmove / memfill / CRC32C / DIF) and **IAA / IAX** (In-Memory Analytics Accelerator — Deflate / Huffman / filter / scan). Both are exposed as PCIe devices with the same SIOV (Scalable I/O Virtualization) architecture: a set of **work-queues (WQs)** + **engines** + **groups** with PASID-aware DMA, **ENQCMD/ENQCMDS** userspace submission, and **completion records** in memory.

The driver implements three personalities for each WQ via the `device_driver` bus model in `bus.c`:

- **`idxd_dma_drv`** (`dma.c`) — kernel-only WQ exposed as a `dma_device` to the dmaengine framework (DMA_MEMCPY only today).
- **`idxd_user_drv`** (`cdev.c`) — userspace WQ exposed as `/dev/dsa/wqX.Y` or `/dev/iax/wqX.Y` character device for `mmap`+`ENQCMD` userspace submit.
- **`idxd_mdev_drv` / iommufd** — VFIO/iommufd passthrough to guests (covered separately).

WQ configuration (engine assignment, threshold, mode, priority, GENCAP gating) is driven by sysfs under `/sys/bus/dsa/devices/wqX.Y/`. INTEL-SA-01084 is the security backdrop: untrusted userspace submission to a WQ requires explicit `user_submission_safe` per-driver-data opt-in and per-platform allowlisting.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct idxd_device` | per-PCIe-function: regs + groups + wqs + engines + EVL + IMS | `drivers::idxd::IdxdDevice` |
| `struct idxd_wq` | per-WQ: type (kernel/user/mdev) + mode (DEDICATED/SHARED) + size + threshold + ats_disable + pasid | `drivers::idxd::Wq` |
| `struct idxd_engine` / `struct idxd_group` | per-engine + per-group topology | `drivers::idxd::{Engine,Group}` |
| `idxd_pci_probe(pdev, id)` | PCI bind, MSI-X alloc, capability read, sysfs registration | `IdxdDevice::probe` |
| `idxd_probe(idxd)` / `idxd_cleanup(idxd)` | per-device bring-up (MSI-X, EVL, INT_HANDLE) | `IdxdDevice::bring_up` / `_tear_down` |
| `idxd_read_caps(idxd)` | CAP/GENCAP/WQCAP/GRPCAP/ENGCAP register decode | `IdxdDevice::read_caps` |
| `idxd_setup_wqs(idxd)` / `_engines` / `_groups` | per-resource alloc + sysfs registration | `IdxdDevice::setup_{wqs,engines,groups}` |
| `idxd_drv_enable_wq(wq)` / `idxd_drv_disable_wq(wq)` | per-WQ enable from sysfs `bind` | `Wq::drv_enable` / `_disable` |
| `idxd_enable_system_pasid(idxd)` / `_disable_system_pasid` | per-device system PASID for kernel-WQ | `IdxdDevice::pasid_{enable,disable}` |
| `idxd_user_drv_probe(idxd_dev)` | `/dev/dsa/wqX.Y` cdev bind + SVA `iommu_sva_bind_device` | `Wq::user_probe` |
| `idxd_cdev_open(inode, filp)` / `_mmap(filp, vma)` / `_poll(...)` | per-WQ user fops: alloc PASID + map portal page | `WqUserCdev::{open,mmap,poll}` |
| `idxd_submit_desc(wq, desc)` | kernel-WQ submit via MOVDIR64B / ENQCMDS | `Wq::submit_desc` |
| `idxd_misc_thread(data)` | EVL (Event Log) drain + per-fault dispatch | `IdxdDevice::evl_thread` |
| `idxd_dma_drv_probe(idxd_dev)` | bind kernel WQ to dmaengine `dma_device` | `Wq::dma_probe` |
| `dsa_completion_record` / `iax_completion_record` | per-descriptor result block (in caller memory) | `drivers::idxd::Completion{Dsa,Iax}` |
| `idxd_wq_disable_pasid(wq)` / `idxd_drv_disable_wq` | per-WQ teardown + PASID drain | `Wq::disable_pasid` |

## Compatibility contract

REQ-1: PCI bind via `idxd_pci_tbl` matches Intel vendor 0x8086 + DSA/IAA device IDs (SPR / GNR-D / DMR platforms); `idxd_driver_data` distinguishes DSA vs IAX type with per-type `compl_size`, `align`, `user_submission_safe`.

REQ-2: Per-device MMIO regions: BAR0 (config registers per VT-d ECAP), BAR2 (per-WQ portal pages — one 4 KB page per WQ for SHARED submission via ENQCMDS, plus one per dedicated WQ for MOVDIR64B).

REQ-3: WQ modes: DEDICATED (single submitter, MOVDIR64B, lower latency) and SHARED (multiple submitters, ENQCMDS, atomic). `user_submission_safe` SHARED-mode WQs are the only userspace-safe surface.

REQ-4: PASID required: kernel WQs use a single "system PASID" allocated via `iommu_alloc_global_pasid`; user WQs each call `iommu_sva_bind_device(dev, mm)` per `idxd_cdev_open` and write the returned PASID into the WQCFG register.

REQ-5: Event Log (EVL): per-device DMA-coherent ring used by HW to log Page Faults, Completion-Record-Address-Fault, and Descriptor-Submission errors when synchronous return-to-submitter isn't possible. Driver thread (`idxd_misc_thread`) drains and dispatches.

REQ-6: Completion records: each submitted descriptor names a `compl_addr` in caller memory (kernel or user PASID space); HW writes status + result there; submitter polls `compl->status` until non-zero (or waits on an interrupt-on-completion if WQCFG enables it).

REQ-7: sysfs WQ-config protocol: write `wq_size`, `mode`, `priority`, `name`, `type` to `/sys/bus/dsa/devices/wqX.Y/` then write `wqX.Y` to `/sys/bus/dsa/drivers/idxd-user/bind` (or `idxd-dmaengine/bind`); driver enables WQ via WQCFG register + WQCMD ENABLE_WQ.

REQ-8: GENCAP-gated features: per-CAP bit checks for ENQCMD/S support, IMS interrupt mode, block-on-fault, MAX_TC, MAX_DESCS_PER_ENGINE; refuse mis-configured WQ that exceeds platform caps.

REQ-9: Hot-reset handling (`idxd_reset_prepare` / `_done`): on PCI FLR, save full per-device + per-WQ config to `idxd_saved_states`, reset HW, restore.

REQ-10: INTEL-SA-01084 conformance: only `user_submission_safe == true` per-WQ-type or platform-allowlisted devices accept user-mode submission; per-WQ `wq->ats_disable` advised on platforms where ATS-PRI interaction is unsafe.

## Acceptance Criteria

- [ ] AC-1: `accel-config list` shows DSA + IAA devices on Sapphire Rapids / Granite Rapids; per-WQ `state`, `size`, `mode`, `priority`, `type` correctly reported.
- [ ] AC-2: Kernel DMA: bind one DSA WQ as `idxd-dmaengine`; `dmatest` runs MEMCPY through it; transferred-byte sysfs counter increments.
- [ ] AC-3: User DSA: bind one DSA WQ as `idxd-user`; `/dev/dsa/wqX.Y` opens; user-mode test (DPDK / accel-config test) submits 1M descriptors via ENQCMDS + sees completion records populated.
- [ ] AC-4: SVA: user binding under a non-root PASID; child of `fork` cannot share PASID; teardown drains all in-flight before PASID release.
- [ ] AC-5: EVL stress: deliberate page-fault on completion-record VA; EVL thread reports + retries / fails correctly without descriptor-loss.
- [ ] AC-6: PCI FLR cycle: trigger FLR via sysfs; per-WQ + per-Group config restored; in-flight descriptors aborted with proper status.
- [ ] AC-7: Misconfig refusal: write `wq_size > max_wqs_total_size` -> sysfs returns `-EINVAL`; bind WQ with unset PASID -> `-EINVAL`.

## Architecture

`IdxdDevice::probe`:

1. `pci_enable_device` + `pci_request_regions` + `pcim_iomap_regions(0,2)` (BAR0 config, BAR2 portals).
2. `idxd_alloc(pdev, data)` zeroes the `idxd_device`, sets `data->type` (`IDXD_TYPE_DSA` or `IDXD_TYPE_IAX`), allocates ida id.
3. `idxd_read_caps(idxd)` decodes GENCAP/WQCAP/GRPCAP/ENGCAP/OPCAP/IAA_CAP registers into `idxd->hw.{gen_cap,wq_cap,grp_cap,eng_cap,op_cap,iaa_cap}`.
4. `idxd_setup_internals` -> `_setup_wqs` (alloc `max_wqs` `idxd_wq` slots + per-WQ portal + per-WQ kobj), `_setup_engines`, `_setup_groups`.
5. `idxd_setup_interrupts` allocates MSI-X (1 misc + N WQ); registers `idxd_misc_thread` for the misc IRQ (handles EVL + INT_HANDLE).
6. If `gen_cap.cmd_cap` and module-param `sva == true`: `iommu_dev_enable_feature(dev, IOMMU_DEV_FEAT_SVA)` + `iommu_dev_enable_feature(dev, IOMMU_DEV_FEAT_IOPF)`; system PASID allocated.
7. `idxd_setup_sysfs(idxd)` registers `/sys/bus/dsa/devices/dsaN/`, `engineN.M/`, `groupN.M/`, `wqN.M/` with per-attribute show/store.
8. `device_register(&idxd->conf_dev)` — main config device; per-resource confdevs registered subsequently.

`Wq::user_probe(idxd_dev)` (per-`/dev/dsa/wqX.Y` `idxd-user` bind):

1. Validate `wq->type == IDXD_WQT_USER` AND `wq->mode == IDXD_WQ_SHARED` AND driver-data `user_submission_safe == true`.
2. `idxd_drv_enable_wq(wq)` programs WQCFG (size, threshold, mode, priority, max_xfer, max_batch, ATS-disable per platform allowlist, op-config bitmap restricting descriptor opcodes).
3. WQCMD ENABLE_WQ -> WQ enters ENABLED state.
4. `idxd_wq_add_cdev(wq)` registers cdev at minor allocated from `file_ida` global; `/dev/dsa/wqN.M` created.
5. Per-open `idxd_cdev_open`:
   - `iommu_sva_bind_device(dev, current->mm)` returns `iommu_sva` + PASID.
   - Allocate `idxd_user_context`, attach `ctx->mm`, `ctx->pasid`, `ctx->sva`, `ctx->task = get_task_struct(current)`.
   - `xa_insert(&wq->upasid_xa, pasid, ctx, GFP_KERNEL)` so EVL faults can find the context.
6. `idxd_cdev_mmap` validates `vma->vm_pgoff` resolves to this WQ's portal page; `io_remap_pfn_range` maps the portal (PROT_WRITE only).
7. Userspace builds a `dsa_hw_desc` (or `iax_hw_desc`), populates PASID + completion-addr + opcode + src/dst, executes `ENQCMDS` against the portal MMIO; HW reads via DMA-PASID, writes completion record via DMA-PASID.

Submission `idxd_submit_desc(wq, desc)` (kernel side, `submit.c`):

1. Acquire per-WQ submission queue slot (idr-allocated, bounded by `wq->size`).
2. Build `dsa_hw_desc` with PASID = system PASID, opcode = MEMMOVE (for DMA_MEMCPY), src/dst as DMA addresses already mapped.
3. For DEDICATED: `movdir64b(portal, desc)` (assembly intrinsic).
4. For SHARED: `enqcmds(portal, desc)` -> retry on submission failure (HW queue full).
5. Caller polls `desc->completion->status != 0` or sleeps on a completion waiter.

EVL drain (`idxd_misc_thread`):

1. Drain EVL ring via MMIO head/tail.
2. Per-entry: extract PASID + WQ + Source-Address + fault-type.
3. Lookup `idxd_user_context` via `xa_load(&wq->upasid_xa, pasid)`.
4. For PRS fault: defer to IOMMU PRQ (cross-ref `intel-iommu.md`); for completion-record-fault: write fault-status into a fresh completion record so userspace can retry; for INT_HANDLE: schedule per-WQ completion notification.
5. Increment per-context fault counters surfaced via sysfs.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `wq_state_machine_no_skip` | INVARIANT | WQ traverses DISABLED -> ENABLED -> DISABLED legally; no double-enable or enable-without-PASID |
| `pasid_xa_uniqueness` | UNIQUENESS | `wq->upasid_xa` never holds two contexts for the same PASID |
| `portal_mmap_bound` | OOB | `idxd_cdev_mmap` rejects offsets outside the WQ's single portal page |
| `evl_drain_no_oob` | OOB | EVL head/tail bounded by ring size; per-iteration drain stops at producer-side head |
| `desc_compl_addr_pasid_validated` | INTEGRITY | submitted descriptor's `compl_addr` is in caller-PASID address space (kernel system PASID for kernel WQ, user PASID for user WQ) |

### Layer 2: TLA+

`models/idxd/wq_lifecycle.tla` (parent-declared): proves WQ enable/disable + driver bind/unbind + cdev open/close + PASID alloc/release sequence cannot leave a stale PASID bound to a freed `idxd_user_context`; reset-prepare/done preserves config invariants.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| Post `idxd_drv_enable_wq`: `wq->state == IDXD_WQ_ENABLED` AND WQCFG matches sysfs-set fields AND PASID populated | `Wq::drv_enable` |
| Post `idxd_user_drv_remove`: `wq->upasid_xa` empty AND every prior `iommu_sva_bind_device` paired with `_unbind` | `Wq::user_remove` |
| Post `idxd_submit_desc`: descriptor either accepted (ENQCMDS success -> queued) or rejected with `-EAGAIN`; never lost | `Wq::submit_desc` |
| Post `idxd_misc_thread` iteration: EVL head advanced to producer head; all entries dispatched | `IdxdDevice::evl_thread` |

### Layer 4: Verus functional

End-to-end: user `open(/dev/dsa/wqX.Y)` -> `mmap` portal -> `ENQCMDS` desc with user-PASID compl addr -> HW DMA -> completion record visible to userspace polling -> `close(fd)` -> SVA unbind -> PASID released after EVL drain quiesce.

## Hardening

INTEL-SA-01084 is the central constraint: the SHARED-WQ + ENQCMDS path is reachable from CPL-3 and writes attacker-influenced descriptors to a HW unit that DMAs through the IOMMU under the submitter's PASID. The driver mitigates via `user_submission_safe` per-driver-data flag (currently `false` on both DSA and IAX), gating user-WQ bind on platform allowlist and per-WQ ATS-disable. Document the allowlist as part of the contract; never elide.

`idxd_cdev_open` calls `iommu_sva_bind_device(dev, current->mm)`; the returned PASID is written into per-WQ WQCFG and used by HW for every subsequent DMA. PASID lifetime is anchored to the cdev `idxd_user_context`; close must drain in-flight descriptors before unbind, otherwise HW DMA continues against a torn-down `mm_struct`. Today the driver uses EVL-drain + `idxd_wq_disable_pasid` + `synchronize_rcu` to fence; capture as functional invariant.

Per-WQ sysfs is CAP_SYS_ADMIN gated by udev-default but the kernel does not enforce. Add explicit `capable(CAP_SYS_ADMIN)` check in `wq_*_store` paths to defeat udev-policy escapes.

`/dev/dsa/wqX.Y` device-node permissions are 0660 root:root by default; the cdev path is the primary surface — treat per-open as RAWIO-class.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `idxd_user_context`, `dsa_hw_desc`, `iax_hw_desc`, `dsa_completion_record`, `iax_completion_record` slabs; user-mode reads of completion records pass through bounded copy_to_user only.
- **PAX_KERNEXEC** — `idxd_user_drv` / `idxd_dma_drv` / `idxd_mdev_drv` `idxd_device_driver.probe`/`remove` in `__ro_after_init`; per-PCI `pci_driver` ops RO.
- **PAX_RANDKSTACK** — randomize stack across `idxd_pci_probe`, `idxd_cdev_open`/`_mmap`/`_release`, `idxd_drv_enable_wq`, `idxd_misc_thread`.
- **PAX_REFCOUNT** — saturating refcount on `idxd_wq.client_count`, `idxd_user_context` ref (used by EVL lookup), `idxd_device.ref` so race between EVL drain and cdev release cannot UAF.
- **PAX_MEMORY_SANITIZE** — zero-on-free `idxd_user_context`, per-WQ portal-page backing, EVL ring entries on drain, saved-state buffers from FLR path so PASIDs + compl-addrs do not bleed across re-bind.
- **PAX_UDEREF** — SMAP/PAN on every `idxd_cdev_*` entry; reject any compl-addr that lies outside the user PASID address space.
- **PAX_RAP / kCFI** — `idxd_device_driver.{probe,remove}`, `idxd_pci_driver.{probe,remove,shutdown,err_handler}`, `idxd_cdev_fops`, and per-bus `idxd_bus_match` typed; vtable in `__ro_after_init`.
- **GRKERNSEC_HIDESYM** — suppress per-WQ portal physical address + completion-record kernel pointer disclosure; gate `/sys/bus/dsa/devices/wqX.Y/portal` and `/sys/kernel/debug/dsa*` behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict idxd probe banners (PCI BDF, MMIO base, MSI-X vectors, GENCAP value disclosing platform stepping) to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict on sysfs** — `wq_*_store`, `engine_*_store`, `group_*_store`, `device_*_store` paths gated by `capable(CAP_SYS_ADMIN)` in the owning user namespace; defeat udev-policy bypass.
- **SVA PASID gating** — `iommu_sva_bind_device` allowed only on `user_submission_safe == true` per-platform allowlist; refuse SVA bind on unsupported steppings; PASID written into WQCFG only after IOMMU PASID-table install completes.
- **`/dev/dsa-*` / `/dev/iax-*` RAWIO class** — cdev open requires CAP_SYS_RAWIO in the owning user namespace; per-open audit trail.
- **mdev / iommufd allowlist** — `idxd_mdev_drv` (covered in iommufd Tier-3) gates VFIO/iommufd bind to explicit platform + WQ allowlist.
- **Completion-record PAX_USERCOPY** — `compl_addr` validated in caller PASID address space; copy_to_user from completion record bounded to `compl_size`; defense against compl-addr poisoning to overwrite arbitrary user memory.
- **EVL ring sanitize** — drained EVL entries zeroed before producer-side reuse; refuse to dispatch entries with malformed `pasid` / `wq_id` fields.
- **WQCFG validation** — pre-program WQCFG validated against GENCAP/WQCAP; refuse impossible config (e.g. `wq_size > max_wqs_total_size`, threshold > size, mode SHARED without ENQCMD support, ATS-enable on non-allowlist platforms).

Rationale: IDXD is the only commodity kernel surface that exposes hardware DMA submission directly to CPL-3 (via ENQCMDS portal MMAP). Every other DMA path the kernel offers is either kernel-only (dmaengine, IOAT) or guest-only (VFIO, virtio). INTEL-SA-01084 is not a theoretical hardening item — it's the platform's published recognition that this surface needs structural gating. CAP_SYS_RAWIO on cdev open, SVA-bind allowlist, completion-record PASID-space validation, mdev/iommufd allowlist, and EVL sanitize collectively transform a per-platform-fragile bind into a structural boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `idxd_mdev_drv` VFIO/iommufd passthrough (covered separately under `vfio/` Tier-3)
- IAA per-opcode (Deflate / Huffman / scan) detail (covered in `idxd-iaa.md` future Tier-3)
- IDXD perfmon details (covered in `idxd-perfmon.md` future Tier-3)
- accel-config userspace (covered in Documentation Tier-3 if any)
- 32-bit-only paths
- Implementation code
