# Tier-3: drivers/block/virtio_blk.c — paravirt block device for KVM/Xen-HVM guests (virtqueue, discard, write-zeroes, secure erase, zoned)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/block/00-overview.md
upstream-paths:
  - drivers/block/virtio_blk.c
  - include/uapi/linux/virtio_blk.h
-->

## Summary

`virtio-blk` is the paravirtualised block device driver used by KVM, Xen-HVM, cloud-hypervisor, and most other virtio-capable hypervisors. Each `virtio_device` of `VIRTIO_ID_BLOCK` becomes a `/dev/vdN` gendisk; the host (hypervisor) implements the actual storage and exchanges descriptors over one or more virtqueues. Requests are encoded as `struct virtio_blk_outhdr` (type + ioprio + sector) followed by a per-request payload scatter-gather list, and replies are a single status byte (`VIRTIO_BLK_S_OK / _IOERR / _UNSUPP / _ZONE_*`). The driver negotiates a feature set with the host (`VIRTIO_BLK_F_*`): SIZE_MAX, SEG_MAX, GEOMETRY, RO, BLK_SIZE, FLUSH, TOPOLOGY, CONFIG_WCE, MQ (multi-queue), DISCARD, WRITE_ZEROES, SECURE_ERASE, ZONED. Multi-queue support exposes up to `num_queues` virtqueues to blk-mq for SMP scaling, with optional `poll_queues` dedicated to IOPOLL mode for low-latency NVMe-style polling.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct virtio_blk` | per-device state (vdev, vdev_mutex, disk, tag_set, config_work, vqs[], num_vqs, zone_sectors) | `drivers::block::virtio_blk::Vblk` |
| `struct virtio_blk_vq` | per-virtqueue state (vq, lock spinlock, name) | `drivers::block::virtio_blk::Vq` |
| `struct virtblk_req` | per-request state (in `request::pdu`): `out_hdr`, `in_hdr` union (status / zone_append in-hdr), `sg_table`, inline sg[] | `drivers::block::virtio_blk::Req` |
| `struct virtio_blk_outhdr` (uapi) | wire request header: `type`, `ioprio`, `sector` | (uapi) |
| `struct virtio_blk_config` (uapi) | per-device config space layout | (uapi) |
| `virtblk_fops` (`block_device_operations`) | `virtblk_open`/`virtblk_release`/`virtblk_getgeo`/`virtblk_freeze`/`virtblk_restore`/`virtblk_report_zones` | `drivers::block::virtio_blk::FOPS` |
| `virtio_mq_ops` (`blk_mq_ops`) | `virtio_queue_rq`, `virtio_queue_rqs`, `virtblk_request_done`, `virtblk_map_queues`, `virtblk_poll`, `virtblk_complete_batch` | `drivers::block::virtio_blk::MQ_OPS` |
| `virtio_blk` (`virtio_driver`) | virtio bus entry: `feature_table`, `id_table = {VIRTIO_ID_BLOCK}`, `probe`, `remove`, `config_changed`, `freeze`/`restore`/`reset_prepare`/`reset_done` | `drivers::block::virtio_blk::Driver` |
| `virtblk_probe` / `virtblk_remove` | per-device init/teardown | `Driver::probe` / `_remove` |
| `init_vq` | per-device virtqueue setup (find_vqs callback) | `Vblk::init_vq` |
| `virtblk_setup_request` / `virtblk_setup_discard_write_zeroes_erase` | per-request header + sg setup | `Vblk::setup_request` / `_setup_discard_write_zeroes_erase` |
| `virtblk_done` / `virtblk_request_done` / `virtblk_complete_batch` | completion path | `Vblk::done` / `_request_done` / `_complete_batch` |
| `virtblk_config_changed` (workqueue handler) | per-device config-space resize handling | `Vblk::config_changed` |
| `virtblk_get_id` | `VIRTIO_BLK_T_GET_ID` (returns 20-byte serial via sysfs `serial`) | `Vblk::get_id` |
| `virtblk_attr_group` (sysfs `cache_type`, `serial`) | per-device attributes | `Vblk::Attrs` |
| `virtblk_freeze_priv` / `virtblk_restore_priv` / `virtblk_reset_prepare` / `virtblk_reset_done` | PM + reset hooks | `Vblk::freeze_priv` / `_restore_priv` / `_reset_prepare` / `_reset_done` |
| `num_request_queues` / `poll_queues` (module params) | virtqueue count limits | `Module::Params` |

## Compatibility contract

REQ-1: Driver matches `id_table = { VIRTIO_ID_BLOCK, VIRTIO_DEV_ANY_ID }`; probes via the virtio bus per `register_virtio_driver`.

REQ-2: Feature negotiation per `features[]` array: SIZE_MAX, SEG_MAX, GEOMETRY, RO, BLK_SIZE, FLUSH, TOPOLOGY, CONFIG_WCE, MQ, DISCARD, WRITE_ZEROES, SECURE_ERASE, ZONED. `features_legacy[]` omits ZONED for legacy transports.

REQ-3: Per-device blk-mq tag set: `nr_hw_queues = num_vqs` (capped by `nr_cpu_ids` and `num_request_queues` module param), `queue_depth = vq's ring length`, `cmd_size = sizeof(struct virtblk_req) + VIRTIO_BLK_INLINE_SG_CNT * sizeof(scatterlist)`, `flags = BLK_MQ_F_SHOULD_MERGE | BLK_MQ_F_STACKING | BLK_MQ_F_BLOCKING (where blocking transports needed)`.

REQ-4: `struct virtio_blk_outhdr` written by guest (driver) into a read-only descriptor; `status` byte written by host into a write-only descriptor; layout differs slightly for `VIRTIO_BLK_T_ZONE_APPEND` (extended in-hdr with `__virtio64 sector + u8 status`).

REQ-5: Request types supported (per VIRTIO 1.2 spec §5.2): `VIRTIO_BLK_T_IN`, `_OUT`, `_FLUSH`, `_GET_ID`, `_DISCARD`, `_WRITE_ZEROES`, `_SECURE_ERASE`, `_ZONE_APPEND`, `_ZONE_REPORT`, `_ZONE_OPEN`, `_ZONE_CLOSE`, `_ZONE_FINISH`, `_ZONE_RESET`, `_ZONE_RESET_ALL`.

REQ-6: Discard/Write-Zeroes/Secure-Erase use `struct virtio_blk_discard_write_zeroes` (sector + num_sectors + flags) ranges; up to `MAX_DISCARD_SEGMENTS = 256` ranges per request, capped by `max_discard_seg` and `max_secure_erase_seg` from config space.

REQ-7: Zoned support (CONFIG_BLK_DEV_ZONED): driver caches `zone_sectors` from `virtio_blk_zoned_characteristics`, registers `report_zones` callback to walk zones via `VIRTIO_BLK_T_ZONE_REPORT`.

REQ-8: Polling (`poll_queues` > 0): per-virtqueue without IRQ; blk-mq HCTX_TYPE_POLL queue calls `virtblk_poll` which calls `virtqueue_more_used` directly.

REQ-9: Batched completion: `virtblk_complete_batch` collects up to `IO_COMP_BATCH_MAX` completions per iteration to amortise per-completion overhead.

REQ-10: PM hooks: `virtblk_freeze`/`virtblk_restore` (CONFIG_PM_SLEEP) and `virtblk_reset_prepare`/`_reset_done` quiesce the queue, reset the device, reinitialise virtqueues.

REQ-11: Sysfs: `/sys/block/vdN/serial` (20-byte virtio device ID), `/sys/block/vdN/cache_type` (writeback/writethrough toggle via `VIRTIO_BLK_F_CONFIG_WCE`).

REQ-12: Module params `num_request_queues=N` (0 = no limit) and `poll_queues=N` are `0644` (runtime tunable at module-load via systemd-modules-load.d).

## Acceptance Criteria

- [ ] AC-1: KVM guest with `-drive file=disk.qcow2,if=virtio` boots and `/dev/vda` is present.
- [ ] AC-2: `cat /sys/block/vda/queue/nr_hw_queues` matches `num_queues` advertised by the host (or `num_request_queues` cap).
- [ ] AC-3: `fstrim /` issues VIRTIO_BLK_T_DISCARD; host observes discards (verifiable via `qemu-img info` showing actual_size shrink).
- [ ] AC-4: `blkdiscard --secure /dev/vda` works on `VIRTIO_BLK_F_SECURE_ERASE`-capable host.
- [ ] AC-5: Zoned host: `blkzone report /dev/vda` enumerates zones; `blkzone reset` round-trips through `VIRTIO_BLK_T_ZONE_RESET`.
- [ ] AC-6: `cat /sys/block/vda/serial` returns the 20-byte device serial obtained via `VIRTIO_BLK_T_GET_ID`.
- [ ] AC-7: `blktests` block/virtio_blk subset passes.
- [ ] AC-8: Polling: `fio --ioengine=libaio --hipri --rw=randread --bs=4k --iodepth=128 --filename=/dev/vda` exercises poll queues and ftrace shows zero IRQ count on poll vqs.

## Architecture

Each `virtio_blk` instance owns one `virtio_device` plus a `virtio_blk::vqs[]` array of `num_vqs` virtqueues. At probe time `init_vq` calls `vdev->config->find_vqs` with a name list (`req.0`, `req.1`, ..., `req.N` for request queues and `req_poll.0`, ... for poll queues) and a callback list (`virtblk_done` for IRQ-driven request queues, NULL for poll queues). `find_vqs` allocates the per-virtqueue rings and binds them to the underlying transport (virtio-pci, virtio-mmio, virtio-ccw).

Data path: `virtio_queue_rq` (or `virtio_queue_rqs` batched) → `virtblk_setup_request` populates `out_hdr.type / ioprio / sector` from `req_op(rq)` and `blk_rq_pos(rq)`, builds the scatter-gather list via `blk_rq_map_sg` into `virtblk_req::sg`, then `virtqueue_add_sgs(vq, sgs, 1, 1, vbr, GFP_ATOMIC)` adds two SG groups (out: out_hdr + data for writes, in: data for reads + in_hdr) to the virtqueue; `virtqueue_kick_prepare + virtqueue_notify` notifies the host. On host completion the host writes the status byte into `vbr->in_hdr.status`, marks the used descriptor, and (for IRQ vqs) raises the configured MSI-X interrupt.

Completion: IRQ vector dispatches `virtblk_done(vq)` which spins consuming used descriptors via `virtqueue_get_buf_ctx` and calling `blk_mq_complete_request_remote` (or `blk_mq_add_to_batch + virtblk_complete_batch` for batching). For poll queues, `virtblk_poll` runs the same loop without IRQ. `virtblk_request_done` is the per-request final callback (called via blk-mq's `mq_ops->complete`), translating `in_hdr.status` to `BLK_STS_*` via `virtblk_result`.

Lock hierarchy: `vblk->vdev_mutex` (per-device, taken on probe/remove/freeze/restore and on every sysfs `_show`/`_store` that touches `vdev`) → `virtio_blk_vq::lock` spinlock (per-virtqueue, taken in `virtio_queue_rq` and in `virtblk_done`). Config-space changes (e.g. resize) raise a host-driven config change interrupt → `virtblk_config_changed` queued on `vblk->config_work` workqueue → `virtblk_update_capacity`.

## Hardening

- `virtblk_setup_request` validates `req_op(rq)` against negotiated features (e.g. DISCARD only if `VIRTIO_BLK_F_DISCARD`); refuses unknown op with `BLK_STS_NOTSUPP`.
- `virtblk_setup_discard_write_zeroes_erase` validates each `(sector, num_sectors)` range against `disksize` and `max_discard_sectors`/`max_secure_erase_sectors` from config space; refuses ranges with overflow.
- `virtblk_complete_batch` and `virtblk_request_done` validate `vbr->in_hdr.status` ∈ {OK, IOERR, UNSUPP, ZONE_*} via `virtblk_result` switch; defaults to `BLK_STS_IOERR` for unknown.
- `VIRTIO_BLK_INLINE_SG_CNT` (2) inlined SGs cover the common case; longer SG lists allocated dynamically via `kmalloc(sizeof(scatterlist) * (nr_sgs+2), GFP_ATOMIC)` and freed in `virtblk_request_done`.
- `virtblk_remove` takes `vdev_mutex`, resets the device via `virtio_reset_device`, sets `vblk->vdev = NULL`, then frees vqs — preventing post-remove DMA.
- Config-changed work flushed in `virtblk_remove` before vdev teardown.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `virtio_blk_config` (kernel-internal, accessed via vio config-space helpers), `serial` sysfs scratch buffer, and `blk_zone[]` arrays returned through `report_zones`; refuse copy_to_user/copy_from_user that touches any other slab.
- **PAX_KERNEXEC** — `virtblk_fops`, `virtio_mq_ops`, `virtio_blk` (virtio_driver), `id_table`, `features[]`/`features_legacy[]`, and `virtblk_attr_group` placed in `__ro_after_init` kernel text.
- **PAX_RANDKSTACK** — randomize kernel stack across `virtio_queue_rq`, `virtblk_done`, `virtblk_request_done`, `virtblk_poll`, `virtblk_get_id`, and `virtblk_config_changed` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `gendisk` holder count, the per-vdev `virtio_device` refcount (via `device::kobj.kref`), and the workqueue config-change reference.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `virtblk_req` slabs (carry `in_hdr.status` + SG arrays), `virtio_blk_vq` slabs (carry virtqueue pointers), and the per-`virtblk_get_id` 20-byte scratch buffer so device IDs and per-request data do not bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs `_store` entry and ioctl entry into the driver; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `virtblk_fops` (`.open`, `.release`, `.getgeo`, `.report_zones`, `.freeze`, `.restore`), `virtio_mq_ops` (`.queue_rq`, `.queue_rqs`, `.complete`, `.map_queues`, `.poll`), and the `virtio_driver` callbacks (`.probe`, `.remove`, `.config_changed`, `.freeze`, `.restore`, `.reset_prepare`, `.reset_done`) marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and `virtio_blk` / `virtqueue` pointer disclosure behind CAP_SYSLOG; suppress `%p` for vq pointers in tracepoints.
- **GRKERNSEC_DMESG** — restrict virtio-blk capacity-change, config-changed, and ZONE_* error banners to CAP_SYSLOG so attackers cannot probe host-side topology via dmesg.
- **Virtqueue descriptor PAX_USERCOPY** — `virtio_blk_outhdr` and `in_hdr` slabs whitelisted; refuse copy_to_user/copy_from_user that touches any other vbr field.
- **Guest-page-pin PAX_REFCOUNT** — pages pinned via `blk_rq_map_sg` reference the underlying bio's page refcount; refcount-overflow saturates rather than wraps; defense against malicious page-aliasing.
- **Discard sector-range check** — every `(sector, num_sectors)` in `virtio_blk_discard_write_zeroes[]` validated against `disksize` AND `max_discard_sectors` AND alignment to `discard_sector_alignment`; refuse overlapping or out-of-range with `BLK_STS_INVAL`.
- **Status-byte whitelist** — `virtblk_result` switch is exhaustive and clamps unknown statuses to `BLK_STS_IOERR`; defense against host injecting unknown status bytes.
- **Zoned in-hdr layout enforcement** — `virtblk_req::in_hdr` union ensures `zone_append.status` is always the last byte; refuse to advance write pointer if the host-returned `zone_append.sector` lies outside the requested zone.
- **Module-param immutability** — `num_request_queues` and `poll_queues` are `0644` but apply only at module load; refuse `add_disk` if values exceed `nr_cpu_ids`.

Rationale: virtio-blk is the kernel's primary "trusted in-VM device but possibly malicious host" boundary — every status byte and every config-space field is hypervisor-controlled and parsed as part of the data path. A relaxed status whitelist, an unvalidated discard range, a writable `virtio_mq_ops`, or a leaked SG slab turns a paravirtualised storage device into a host-to-guest escalation primitive (and vice versa for guest-to-host via crafted descriptors). RAP/kCFI on every vtable, PAX_USERCOPY on descriptor slabs, SIZE_OVERFLOW on every host-supplied count, refcount-overflow trap on bio pages, MEMORY_SANITIZE on per-request slabs, and config-changed work flush at remove turn virtio-blk from "trust the host" into a structural enforcement boundary.

## Open Questions

- [ ] Q1: Should the driver enforce a per-device upper bound on `num_queues` lower than `nr_cpu_ids` to defeat resource-exhaustion DoS by a malicious host advertising 1024 queues?
- [ ] Q2: How should `VIRTIO_BLK_T_ZONE_RESET_ALL` interact with `BLKZEROOUT` ioctl semantics — should the guest synthesise individual zone resets to keep the per-zone audit trail granular?
- [ ] Q3: Should `VIRTIO_BLK_F_SECURE_ERASE` require a guest-side CAP_SYS_ADMIN check on `blkdiscard --secure` since the operation is irrevocable at host-storage level?

## Verification

- `lsblk -d /dev/vda` shows correct major:minor, size, model = "QEMU HARDDISK" (or hypervisor-specific).
- `cat /sys/block/vda/{serial,cache_type}` returns plausible values.
- `cat /sys/block/vda/queue/{nr_hw_queues,queue_depth,logical_block_size,max_hw_sectors_kb,discard_max_bytes,write_zeroes_max_bytes}`.
- `blktests` block/virtio_blk subset passes.
- `dmesg | grep -i virtblk` after probe shows feature negotiation banner.
- ftrace `block:block_rq_issue` and `virtio:*` events flow while running fio against `/dev/vda`.
- `fstrim -v /` returns non-zero trimmed bytes on a discard-capable host.
