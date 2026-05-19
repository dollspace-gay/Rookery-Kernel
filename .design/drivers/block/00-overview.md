# Tier-3: drivers/block/ — block driver subsystem overview (loop, nbd, virtio-blk, null-blk, zram, brd, rbd, xen-blk, ublk)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/block/Kconfig
  - drivers/block/Makefile
  - drivers/block/loop.c
  - drivers/block/nbd.c
  - drivers/block/virtio_blk.c
  - drivers/block/null_blk/main.c
  - drivers/block/zram/zram_drv.c
  - drivers/block/brd.c
  - drivers/block/rbd.c
  - drivers/block/ublk_drv.c
  - drivers/block/xen-blkfront.c
  - drivers/block/xen-blkback/
  - drivers/block/rnbd/
  - drivers/block/aoe/
  - drivers/block/drbd/
  - include/linux/blk-mq.h
  - block/blk-mq.c
-->

## Summary

`drivers/block/` hosts the body of standalone block drivers that do not belong to a more specialised pillar — meaning everything that is not NVMe (`drivers/nvme/`), SCSI (`drivers/scsi/`), MMC (`drivers/mmc/`), MTD (`drivers/mtd/`), or device-mapper (`drivers/md/`). Each driver here exposes a `gendisk` to the block layer via `add_disk`, registers a `blk_mq_tag_set` with `blk-mq` core (`block/blk-mq.c`), and plumbs `struct blk_mq_ops::queue_rq` to translate `struct request` into device-specific I/O — be it a file-backed loop op, a socket send to an NBD server, a virtqueue descriptor, an in-memory copy, or a compressed page pool.

This Tier-3 family covers the six core drivers most useful for containers, virtualisation, testing, and memory compression: `loop.c` (file-backed loopback), `nbd.c` (network block device + netlink configuration), `virtio_blk.c` (paravirt block device for KVM/Xen-HVM guests), `null_blk/` (zero-cost blk-mq test target with zoned and fault-injection support), `zram/` (compressed RAM-backed swap and tmpfs backing), and supplementary drivers (`brd.c` RAM disk, `rbd.c` Ceph RADOS block device, `ublk_drv.c` userspace block driver, `xen-blkfront/back`, `rnbd/`, `aoe/`, `drbd/`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct gendisk` | per-disk control block (registered via `add_disk`) | `drivers::block::Disk` |
| `struct blk_mq_tag_set` | shared tag set across one or more `request_queue`s | `drivers::block::TagSet` |
| `struct blk_mq_ops` | per-driver request-handling vtable | `drivers::block::MqOps` |
| `struct block_device_operations` | per-device fops (open/release/ioctl) | `drivers::block::BlockDevOps` |
| `struct request` / `struct bio` | per-IO block layer carrier (cross-ref `block/00-overview.md`) | (block core) |
| `add_disk(disk)` / `del_gendisk(disk)` | register/unregister a `gendisk` | `Disk::register` / `_unregister` |
| `blk_mq_alloc_tag_set(set)` | allocate tag set | `TagSet::alloc` |
| `blk_mq_alloc_disk(set, lim, data)` | alloc `gendisk` bound to a tag set | `TagSet::alloc_disk` |
| `blk_mq_complete_request(rq)` | mark per-request done | `Request::complete` |
| `__register_blkdev(major, name, probe)` / `unregister_blkdev` | per-major static dev_t reservation | `BlockMajor::register` / `_unregister` |
| `register_virtio_driver(drv)` / `unregister_virtio_driver(drv)` | virtio bus registration (virtio-blk) | `VirtioBus::register_driver` |
| `genl_register_family(family)` / `genl_unregister_family(family)` | netlink genl (nbd) | `Genl::register_family` |
| `misc_register(misc)` / `misc_deregister(misc)` | misc-char node (loop-control) | `Misc::register` |
| `disk_uevent(disk, KOBJ_CHANGE)` | hotplug uevent | `Disk::uevent` |
| `set_disk_ro(disk, ro)` | mark read-only | `Disk::set_ro` |
| `blk_queue_max_hw_sectors`, `blk_queue_logical_block_size`, `blk_queue_io_min`, `blk_queue_io_opt` | per-queue limit setters | `RequestQueue::limits_*` |
| `bdev_open` / `bdev_release` (cross-ref `block/bdev.md`) | per-bdev open/close | (block core) |
| `CONFIG_BLK_DEV_LOOP` / `_NBD` / `_VIRTIO_BLK` / `_NULL_BLK` / `_ZRAM` | per-driver Kconfig gates | `Subsystem::config` |

## Compatibility contract

REQ-1: Each driver registers exactly one `blk_mq_tag_set` per logical device-set (loop: per-loop; nbd: per-nbd; virtio-blk: per-vblk; null-blk: per-nullb or globally shared; zram: per-zram), validated against `nr_hw_queues`, `queue_depth`, and `cmd_size` derived from `sizeof(per-op cmd)`.

REQ-2: Each driver provides a `block_device_operations` table with at minimum `.owner = THIS_MODULE` and per-driver `.open`, `.release`, `.ioctl` as appropriate; `.compat_ioctl` aliases the 64-bit handler where structures are compat-clean.

REQ-3: Each driver's `blk_mq_ops::queue_rq` returns `BLK_STS_OK` on accept and a `BLK_STS_*` on rejection; never blocks the submission path for IO-completion (must defer to workqueue, IRQ, or AIO/netlink callback).

REQ-4: Per-`request` driver state lives inside the request's `pdu` area (`blk_mq_rq_to_pdu`) sized by `tag_set.cmd_size` — no per-IO `kmalloc` on the hot path.

REQ-5: Block dev majors: LOOP_MAJOR (7), NBD_MAJOR (43), `virtblk` (dynamic via `register_blkdev(0, ...)`), null-blk (dynamic), ZRAM_MAJOR (251 by default, dynamic if unavailable).

REQ-6: ioctl numbers in `include/uapi/linux/loop.h` (`LOOP_*`, `LOOP_CTL_*`), `include/uapi/linux/nbd.h` (`NBD_*`), `include/uapi/linux/nbd-netlink.h` (genl commands), `include/uapi/linux/virtio_blk.h` (feature bits + request types); zero ABI drift permitted at the syscall boundary.

REQ-7: sysfs surfaces under `/sys/block/<dev>/`: per-driver attributes (loop: `backing_file`, `offset`, `sizelimit`, `autoclear`, `partscan`, `dio`; nbd: `pid`, `backend`; zram: `disksize`, `compact`, `reset`, `comp_algorithm`, `backing_dev`, `writeback`, `mm_stat`, `bd_stat`, `io_stat`); read paths gated by capabilities where exposing kernel data.

REQ-8: configfs surface under `/sys/kernel/config/nullb/` is provided exclusively by `null_blk`; it MUST refuse to expose `data`, `cache`, or in-flight `nullb_zone` state to non-CAP_SYS_ADMIN callers.

REQ-9: Module-load order: drivers register `gendisk`s lazily — pre-creation of N devices (loop, null-blk, zram) is controlled by `max_loop`, `nr_devices`, `num_devices` module params, all `0444` (read-only after load).

REQ-10: Per-driver removal/teardown order: stop accepting new IO via `blk_mq_freeze_queue` → `del_gendisk` → drain in-flight via `blk_mq_quiesce_queue` → free `tag_set` → free per-device state; no shortcut paths.

REQ-11: blk-cgroup integration: drivers that proxy IO to a backing file (loop) MUST capture per-bio `blkcg_css` + `memcg_css` and reapply them on the kthread/workqueue executor so cgroup throttling and memcg accounting are not bypassed.

REQ-12: Zoned-device support (null-blk zoned, virtio-blk zoned): implement `report_zones`, `BLK_OP_ZONE_*` handlers per `Documentation/block/zoned.rst` semantics with per-zone state machine.

## Acceptance Criteria

- [ ] AC-1: `lsblk` enumerates every loaded driver's devices with correct major:minor, size, and `MODEL=`.
- [ ] AC-2: `blktests` block/* suite (https://github.com/osandov/blktests) passes for each driver with default config.
- [ ] AC-3: `mkfs.ext4 + mount + dd + fsck` cycle works through each driver (loop on tmpfs, nbd against `nbd-server`, virtio-blk in qemu, null-blk memory-backed, zram).
- [ ] AC-4: `cat /sys/block/<dev>/queue/*` reports plausible `nr_hw_queues`, `queue_depth`, `logical_block_size`, `max_sectors_kb` per driver.
- [ ] AC-5: ioctl ABI test: `loop-control` LOOP_CTL_GET_FREE / LOOP_CTL_ADD / LOOP_CTL_REMOVE round-trip cleanly; `NBD_DO_IT` ioctl path returns when nbd-server disconnects.
- [ ] AC-6: zoned tests (`blkzone report`, `blkzone reset`) work against null-blk-zoned and virtio-blk-zoned.

## Architecture

The `drivers/block/` family is structurally simple: each driver is a thin shim that (1) discovers or accepts a backing store (a file, a socket, a virtqueue, a memory area, a compressed page pool), (2) allocates a `gendisk` + `blk_mq_tag_set`, (3) implements `queue_rq` to translate `struct request` into a backing-store operation, and (4) exposes a configuration plane (ioctl, netlink genl, sysfs, configfs). The block layer (`block/blk-mq.c`) handles scheduling, tag allocation, request queuing, completion batching, and the request_queue lifecycle — drivers do not implement any of that themselves.

The data path is uniform: `submit_bio` → `blk-mq` dispatch picks a `blk_mq_hw_ctx` → driver `queue_rq` is called with a started request → driver enqueues work (file IO, socket send, virtqueue add+kick, memcpy, compression) → completion (workqueue callback / IRQ / kthread / AIO completion / netlink reply) calls `blk_mq_complete_request(rq)`. The driver's `.complete` callback in `blk_mq_ops` runs in softirq or the completion CPU and finalises per-request state.

Lock hierarchy is per-driver but always rooted at `request_queue::queue_lock` (rare — most drivers avoid taking it directly) and the driver-internal config mutex (`lo_mutex`, `nbd->config_lock`, `vblk->vdev_mutex`, `zram->dev_lock`). Module-globals (`loop_index_idr`, `nbd_index_idr`, `nullb_indexes`) are protected by separate global mutexes (`loop_ctl_mutex`, `nbd_index_mutex`) acquired only on add/remove paths.

Driver lifetime is reference-counted through `gendisk::part0.bd_holder_dir` plus per-driver refcounts (`nbd_device::refs`, `nbd_device::config_refs`, etc.). Module unload is gated by these refcounts — drivers cannot unload while devices are open.

## Hardening

- Every driver MUST set `cmd_size` correctly so `blk_mq_rq_to_pdu` never returns an out-of-bounds pointer.
- All `copy_from_user` of `loop_info64`, `loop_config`, `nbd_*` netlink attrs go through bounded slab-backed scratch buffers, never directly into kernel state.
- ioctl handlers validate `cmd` against the driver's allowlist before dispatch; unknown ioctls return -ENOTTY (not -EINVAL or -ENOSYS where the syscall ABI requires -ENOTTY for block devices).
- All netlink genl handlers (`nbd_genl_*`) check `genl_info::nlhdr->nlmsg_pid` ownership where state mutation is involved.
- Per-driver workqueues are bounded (`WQ_PERCPU`, `WQ_UNBOUND` with `max_active`), preventing per-IO-storm DoS.
- Freeze/quiesce order at remove: `del_gendisk` → `blk_mq_freeze_queue` → `blk_mq_quiesce_queue` → `blk_mq_free_tag_set` — no shortcut paths.
- Module params (`max_loop`, `nbds_max`, `nr_devices`) are `0444`; cannot be retuned at runtime to amplify resource use.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `struct loop_info64`, `struct loop_config`, `struct nbd_request`, `struct nbd_reply`, `struct virtio_blk_outhdr`, `struct virtio_blk_discard_write_zeroes`, and zram sysfs scratch buffers; refuse copy_to_user/copy_from_user that touches any other slab.
- **PAX_KERNEXEC** — all block-driver code lives in W^X kernel text; per-driver `blk_mq_ops`, `block_device_operations`, `virtio_driver`, and `genl_family` tables placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel stack across every ioctl entry (`lo_ioctl`, `nbd_ioctl`, `loop_control_ioctl`), netlink genl entry (`nbd_genl_connect`), and `queue_rq` entry.
- **PAX_REFCOUNT** — saturating `refcount_t` on `nbd_device::refs` + `nbd_device::config_refs`, `gendisk` holder count, virtio-blk vdev refcount, zram backing-dev file refcount.
- **PAX_MEMORY_SANITIZE** — zero-on-free for loop encryption-key buffers (legacy crypto path), nbd socket-state slabs, virtio-blk in-hdr buffers, and zram per-slot handles so backing data does not bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every ioctl/genl/sysfs-store entry into the block drivers; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — every `blk_mq_ops` (`queue_rq`, `complete`, `init_hctx`, `exit_hctx`, `poll`, `timeout`), `block_device_operations` (`open`, `release`, `ioctl`, `compat_ioctl`, `free_disk`), and `genl_small_ops::doit` vtable marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-driver pointer disclosure (e.g. `loop_device` addresses, `nbd_device` indices in dmesg) behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict driver fault, IO-timeout, and config-failure dmesg banners to CAP_SYSLOG so attackers cannot probe backing-store layout via dmesg.
- **CAP_SYS_ADMIN at config plane** — `LOOP_CTL_ADD`/`LOOP_CTL_REMOVE`, every `NBD_*` netlink genl op, every zram sysfs store (`disksize`, `reset`, `backing_dev`, `writeback`), and null-blk configfs mkdir/rmdir require CAP_SYS_ADMIN in the device's user namespace.
- **Backing-store FD trust boundary** — loop `LOOP_SET_FD`/`LOOP_CONFIGURE` and nbd `NBD_SOCK_SET`/`NBD_ATTR_SOCKETS` validate the passed FD's owner credentials against the calling task's namespace before binding; refuse to bind an FD owned by a different user namespace.
- **Per-driver workqueue concurrency cap** — `loop` per-cgroup worker tree, `nbd` `recv_workq`, `virtblk_wq`, zram writeback workqueue all bounded with `max_active`; defense against worker-storm DoS.
- **Module-param immutability** — `max_loop`, `nbds_max`, `nr_devices` exposed `0444` (read-only after load); refuse runtime retuning.
- **Refcount-overflow trap** — saturating `refcount_t` on `nbd_device::config_refs` and `nbd_device::refs` defeats double-disconnect/connect races.
- **Read-only ops-tables** — `lo_fops`, `nbd_fops`, `virtblk_fops`, `null_ops`, `zram_devops`, `loop_mq_ops`, `nbd_mq_ops`, `virtio_mq_ops`, `null_mq_ops`, `nbd_genl_family.small_ops` all `__ro_after_init` + kCFI typed.

Rationale: block drivers expose a wide configuration surface (ioctl, genl, sysfs, configfs, module params) and accept FDs and sockets from userland that they then DMA, splice, or copy on behalf of. A relaxed refcount, a missed namespace check on a passed FD, a writable ops-table, or an indirect-call site without kCFI converts a userland-trusted submission into kernel-level data corruption or privilege escalation. Saturating refcounts, kCFI on every vtable, namespace-credential checks on FD-passing, CAP_SYS_ADMIN gating on every config plane, and HIDESYM on dmesg turn the block-driver subsystem from "configurable shared resource" into a structural enforcement boundary.

## Open Questions

- [ ] Q1: Should Rookery merge `null_blk` configfs and `zram` sysfs into a single unified per-driver control plane (configfs everywhere), or preserve historical Linux divergence?
- [ ] Q2: How should the loop-on-fuse case be policed at the kCFI boundary — passing a fuse-backed FD into LOOP_SET_FD creates a recursion path between block and fuse that needs explicit cycle detection.
- [ ] Q3: Are the ublk userspace-driven block driver semantics (`drivers/block/ublk_drv.c`) covered by the same trust model as nbd, or do they require a distinct Tier-3 with their own kCFI typeset?

## Verification

- Per-driver `lsblk` enumeration smoke test.
- `blktests` block/* across loop, nbd, null-blk, virtio-blk.
- `cat /sys/block/<dev>/queue/{logical_block_size,max_hw_sectors_kb,nr_hw_queues,queue_depth}` sanity.
- `dmesg | grep -E '(loop|nbd|virtblk|null_blk|zram)'` after each module load and device add.
- ftrace `block:*` tracepoints exercised while each driver is active.

## Out of Scope

- block layer core / blk-mq scheduler (covered in `block/` Tier-3 family).
- NVMe (`drivers/nvme/`), SCSI (`drivers/scsi/`), MMC/SD (`drivers/mmc/`), MTD (`drivers/mtd/`).
- Device-mapper (`drivers/md/`) and its targets.
- Filesystem-level concerns (covered in `fs/` Tier-3 family).
- Per-vendor RAID controller drivers in `drivers/scsi/`.
