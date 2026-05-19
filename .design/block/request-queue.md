# Tier-3: block/request-queue — request_queue + gendisk + bdev + flush + sysfs + stats

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - block/blk-core.c
  - block/blk-flush.c
  - block/blk-settings.c
  - block/blk-sysfs.c
  - block/blk-rq-qos.c
  - block/blk-stat.c
  - block/blk-pm.c
  - block/blk-ia-ranges.c
  - block/genhd.c
  - block/bdev.c
  - block/fops.c
  - block/bsg.c
  - block/bsg-lib.c
  - block/badblocks.c
  - include/linux/blkdev.h
  - include/uapi/linux/fs.h
-->

## Summary
Tier-3 design for the per-block-device infrastructure: `request_queue` (per-device submission state), `gendisk` (per-block-device identity), `block_device` / `bdev` (block-device file in /dev), flush + write-back synchronization, sysfs queue-tunables, request-queue QoS framework, per-request statistics, runtime PM integration, I/O accounting ranges, BSG (Block-SCSI Generic) ioctl pass-through, and bad-block tracking.

Sub-tier-3 of `block/00-overview.md`. Every block device user-visible interaction (mount, open /dev/sda, ioctl, sysfs query) routes through this Tier-3.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Request-queue core (alloc, init, lifecycle) | `block/blk-core.c` |
| Flush + write-barrier sequencing | `block/blk-flush.c` |
| Per-queue settings (`max_sectors`, `max_segments`, ...) | `block/blk-settings.c` |
| sysfs queue tunables | `block/blk-sysfs.c` |
| Request-queue QoS framework (rq-qos: wbt, ioc, iolatency) | `block/blk-rq-qos.c` |
| Per-request statistics + bucketing | `block/blk-stat.c` |
| Runtime PM integration (suspend/resume of block devices) | `block/blk-pm.c` |
| Independent Access Ranges (multi-actuator NVMe) | `block/blk-ia-ranges.c` |
| gendisk lifecycle | `block/genhd.c` |
| block_device + bdev | `block/bdev.c`, `block/fops.c` |
| BSG (Block-SCSI Generic ioctl pass-through) | `block/bsg.c`, `block/bsg-lib.c` |
| Bad-block tracking (per-disk corrupted-LBA list) | `block/badblocks.c` |
| Public types | `include/linux/blkdev.h` (`struct request_queue`, `struct gendisk`) |
| UAPI ioctls | `include/uapi/linux/fs.h` (BLKROGET / BLKGETSIZE / BLKDISCARD / etc.) |

## Compatibility contract

### `struct request_queue` layout

`include/linux/blkdev.h` defines `struct request_queue` (~600 bytes; ~80 fields). First-cache-line + commonly-accessed fields layout-equivalent. Fields: `last_merge`, `elevator`, `tag_set`, `mq_ops`, `queue_lock`, `mq_freeze_lock`, `q_usage_counter`, `pm_only`, `id`, `disk`, `sysfs_dir_lock`, `kobj`, `mq_kobj`, `pm_runtime_*`, `node`, `srcu`, `nr_requests`, `dma_pad_mask`, `dma_alignment`, `nr_zones`, `conv_zones_bitmap`, `seq_zones_wlock`, `seq_zones_bitmap`, `crypto_profile`, `mq_freeze_depth`, `bsg_dev`, `xt`, `requeue_list`, `requeue_lock`, `requeue_work`, `mq_freeze_wq`, `q_usage_counter_lock`, `last_merge_lock`, `quiesce_depth`, `mq_freeze_depth_lock`.

Layout-equivalent.

### `struct gendisk` layout

`include/linux/blkdev.h` (`struct gendisk`):
- `major`, `first_minor`, `minors` (legacy device-number)
- `disk_name[DISK_NAME_LEN]`
- `flags` (GENHD_FL_*)
- `events`, `event_flags`
- `part0` (struct block_device for whole disk)
- `fops` (block_device_operations)
- `queue` (request_queue back-pointer)
- `private_data`
- `node_id`, `nr_zones`
- `state`, `policy`, `slave_dir`, `slave_dir_kobj`, `integrity_kobj`
- `random`, `events_kthread`, `events_work`, `event_lock`
- `events_async`, `ev_pending`, `ev_block`

Layout-equivalent.

### `struct block_device` layout

`include/linux/blk_types.h`:
- `bd_dev` (dev_t)
- `bd_openers` (refcount)
- `bd_inode` (inode in bdev's pseudo-FS)
- `bd_super`, `bd_holder`, `bd_holders`, `bd_holder_lock`, `bd_holder_dir`, `bd_writers`
- `bd_queue` (request_queue)
- `bd_meta_info`, `bd_disk`, `bd_partno`, `bd_size_lock`
- `bd_device` (struct device)

Layout-equivalent.

### sysfs `/sys/block/<dev>/queue/*`

Per-queue tunables. Format-identical:
- `nr_requests` — max in-flight requests
- `read_ahead_kb` — readahead window
- `max_sectors_kb` — max request size (in KB)
- `max_segments` — max scatter-gather segments per request
- `max_segment_size` — max bytes per segment
- `max_discard_segments` — max discard segments
- `discard_max_bytes`, `discard_max_hw_bytes`, `discard_granularity`, `discard_zeroes_data`
- `physical_block_size`, `logical_block_size`, `optimal_io_size`, `minimum_io_size`
- `rotational` (HDD vs SSD hint)
- `nomerges` — merge-policy (0=any, 1=simple, 2=none)
- `add_random` — entropy contribution
- `iosched`, `scheduler` — current + available schedulers
- `stable_writes`, `iostats`, `dma_alignment`
- `wbt_lat_usec`, `io_poll`, `io_poll_delay`
- `zoned`, `nr_zones`, `chunk_sectors`, `zone_append_max_bytes`, `zone_write_granularity`
- `crypto/*` (per blk-crypto, cross-ref `block/blk-crypto.md`)
- `integrity/*` (per bio-integrity)
- `independent_access_ranges/*` (multi-actuator NVMe)

### sysfs `/sys/block/<dev>/*` (top-level)

`size`, `ro`, `removable`, `ext_range`, `range`, `dev`, `events`, `events_async`, `events_poll_msecs`, `inflight`, `discard_alignment`, `alignment_offset`, `capability`, `stat`, `bdi/*`, `holders/*`, `slaves/*`, `<partition>/*`, `queue/*`, `device/*`. Format-identical.

### Block ioctls (`include/uapi/linux/fs.h`)

`BLKROSET=0x125D`, `BLKROGET=0x125E`, `BLKRRPART=0x125F`, `BLKGETSIZE=0x1260`, `BLKFLSBUF=0x1261`, `BLKRASET=0x1262`, `BLKRAGET=0x1263`, `BLKFRASET=0x1264`, `BLKFRAGET=0x1265`, `BLKSECTSET=0x1266`, `BLKSECTGET=0x1267`, `BLKSSZGET=0x1268`, `BLKBSZGET=0x80081270`, `BLKBSZSET=0x40081271`, `BLKGETSIZE64=0x80081272`, `BLKDISCARD=0x1277`, `BLKIOMIN=0x1278`, `BLKIOOPT=0x1279`, `BLKALIGNOFF=0x127a`, `BLKPBSZGET=0x127b`, `BLKDISCARDZEROES=0x127c`, `BLKSECDISCARD=0x127d`, `BLKROTATIONAL=0x127e`, `BLKZEROOUT=0x127f`, `BLKGETDISKSEQ=0x80081280`, `BLKREPORTZONE`, `BLKRESETZONE`, `BLKGETZONESZ`, `BLKGETNRZONES`, `BLKZNAMENT*`, `BLKOPENZONE`, `BLKCLOSEZONE`, `BLKFINISHZONE`, `BLKAUTOLOAD`, `BLKAUTOUNLOAD`, `BLKBSZ*`, `BLKRRPART`. Numeric byte-identical.

### `/proc/diskstats`

Format-identical: `<major> <minor> <name> <reads_completed> <reads_merged> <sectors_read> <ms_reading> <writes_completed> <writes_merged> <sectors_written> <ms_writing> <ios_in_progress> <ms_io> <weighted_ms_io> [discard fields]`.

### `/proc/partitions`

Format-identical: `<major> <minor> <#blocks> <name>`.

## Requirements

- REQ-1: `struct request_queue` first-cache-line layout-equivalent.
- REQ-2: `struct gendisk` + `struct block_device` first-cache-line layouts equivalent.
- REQ-3: Block ioctl numbers byte-identical; `struct hd_geometry`, `struct hd_driveid` byte-identical.
- REQ-4: sysfs `/sys/block/<dev>/queue/*` content format-identical for every documented tunable.
- REQ-5: `/proc/diskstats` + `/proc/partitions` content format-identical.
- REQ-6: Request-queue lifecycle: `blk_alloc_queue` → `add_disk` (registers + creates sysfs) → use → `del_gendisk` (cleanup) → `blk_cleanup_queue`. Identical state machine.
- REQ-7: Flush + write-barrier semantics: REQ_PREFLUSH/REQ_FUA per upstream's `blk_flush_complete_seq`. Identical sequencing.
- REQ-8: rq-qos framework: wbt + ioc + iolatency; identical hooks (`rq_qos_throttle`, `rq_qos_done`, `rq_qos_track`, `rq_qos_done_bio`).
- REQ-9: Per-request statistics: `blk_rq_io_stat` collects per-request elapsed time, queue time, etc. Identical bucketing.
- REQ-10: Runtime PM: per-block-device runtime suspend/resume per upstream's `blk_pm_*` integration.
- REQ-11: Independent Access Ranges: per-disk multi-actuator support via `blk-ia-ranges.c`; sysfs `independent_access_ranges/*` exposed.
- REQ-12: BSG (SCSI ioctl pass-through): `/dev/bsg/<dev>` userspace interface preserved per upstream.
- REQ-13: Bad-block tracking: per-gendisk bad-block list via `badblocks_set`/`badblocks_check`; userspace via `/sys/block/<dev>/badblocks`.
- REQ-14: TLA+ co-ownership of mandatory L2 models from blk-mq.md (request_state_machine).
- REQ-15: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct request_queue`, `struct gendisk`, `struct block_device` byte-identical first-cache-line. (covers REQ-1, REQ-2)
- [ ] AC-2: A test exercising every BLK* ioctl on /dev/null_blk0 produces byte-identical responses vs. upstream. (covers REQ-3)
- [ ] AC-3: A diff of `cat /sys/block/null_blk0/queue/*` after curated workload byte-identical. (covers REQ-4)
- [ ] AC-4: `cat /proc/diskstats` and `cat /proc/partitions` after curated workload byte-identical. (covers REQ-5)
- [ ] AC-5: `add_disk` + `del_gendisk` round-trip on null_blk preserves correct refcount. (covers REQ-6)
- [ ] AC-6: A REQ_PREFLUSH+REQ_FUA write test produces correct flush sequence on a barrier-aware device. (covers REQ-7)
- [ ] AC-7: `wbt_lat_usec` sysctl and observable wbt throttle behavior on heavy write workload byte-identical. (covers REQ-8)
- [ ] AC-8: A `bcc-tools` `biolatency` script produces upstream-equivalent histograms. (covers REQ-9)
- [ ] AC-9: `pm-suspend` + `pm-resume` cycle on a runtime-PM-aware NVMe device works correctly. (covers REQ-10)
- [ ] AC-10: A multi-actuator NVMe device's `/sys/block/<dev>/queue/independent_access_ranges/*` lists ranges. (covers REQ-11)
- [ ] AC-11: `bsg-tools` `bsg_*` userspace tools work unmodified against /dev/bsg/<dev>. (covers REQ-12)
- [ ] AC-12: A `badblocks` test on a disk with curated corruption returns expected bad-block list via /sys/block/<dev>/badblocks. (covers REQ-13)
- [ ] AC-13: TLA+ models inherited from blk-mq.md pass. (covers REQ-14)
- [ ] AC-14: Hardening section present and follows template. (covers REQ-15)

## Architecture

### Rust module organization

- `kernel::block::queue::RequestQueue` — `request_queue` wrapper
- `kernel::block::queue::settings` — per-queue tunables
- `kernel::block::queue::sysfs` — /sys/block/<dev>/queue/*
- `kernel::block::queue::flush::Flush` — REQ_PREFLUSH/REQ_FUA sequencing
- `kernel::block::queue::rq_qos::{Wbt, Ioc, Iolatency}` — per-queue QoS
- `kernel::block::queue::stat::Stat` — per-request stats
- `kernel::block::queue::pm` — runtime PM
- `kernel::block::queue::ia_ranges` — Independent Access Ranges
- `kernel::block::gendisk::GenDisk` — `gendisk` wrapper
- `kernel::block::bdev::BlockDevice` — `block_device` wrapper
- `kernel::block::bdev::fops` — block_device_operations
- `kernel::block::bsg::Bsg` — SCSI ioctl pass-through
- `kernel::block::badblocks::BadBlocks` — bad-block tracking

### Locking and concurrency

- **`q->queue_lock`** (raw_spinlock; legacy; rare in mq-only path)
- **`q->mq_freeze_lock`** (mutex): per-queue freeze for elevator changes
- **`q->q_usage_counter`** (percpu_ref): in-flight request tracking; freeze drains
- **Per-`bdev` lock**: per-bdev spinlock for state changes (open/close)

### Error handling

- `Err(EBUSY)` — queue in use; cannot teardown
- `Err(EINVAL)` — bad tunable value
- `Err(ENOSPC)` — disk full (passed up from driver)
- `Err(EROFS)` — read-only mount/device
- `Err(ENXIO)` — device gone (hot-unplug)
- `Err(EOPNOTSUPP)` — operation not supported by device

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Request-queue alloc + init | `kani::proofs::block::queue::alloc_safety` |
| Flush sequence state machine | `kani::proofs::block::queue::flush_safety` |
| rq-qos hook dispatch | `kani::proofs::block::queue::rq_qos_safety` |
| Bad-block list manipulation | `kani::proofs::block::badblocks::list_safety` |

### Layer 2: TLA+ models

(Inherits from `block/blk-mq.md`'s mandatory L2 models.)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Request queue holders list | Every block_device with bd_holder set is on the queue's holders list | `kani::proofs::block::bdev::holders_invariants` |
| Bad-block tree | Bad-block ranges are non-overlapping, address-ordered | `kani::proofs::block::badblocks::tree_invariants` |
| Flush sequence | Each flush op (PRE/DATA/POST) fires at most once per request | `kani::proofs::block::queue::flush_seq_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Flush sequence correctness** via Verus — proves: under any sequence of REQ_PREFLUSH/REQ_FUA writes, the device receives the upstream-defined flush sequence.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | request_queue, gendisk, block_device refcounts use `Refcount` | § Mandatory |
| **AUTOSLAB** | per-driver request_queue + gendisk slab caches per-type-tagged | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **UDEREF**: ioctl args from userspace via `UserPtr<...>`
- **SIZE_OVERFLOW**: sector arithmetic uses checked operators; offset math saturating
- **CONSTIFY**: per-driver block_device_operations vtable static const

### Row-2 / GR-RBAC integration

LSM hooks: `security_file_open` (cross-ref `fs/vfs/file-table.md`) for /dev/<dev>; `security_path_chmod` for ioctl. GR-RBAC's policy can deny block-device opens.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — every ioctl carrying user buffers (`BLKGETSIZE64`, `BLKDISCARD`, `BLKZEROOUT`, `BLKTRACESETUP`) bounded; `UserPtr<...>` typed accessor enforced.
- **PAX_KERNEXEC** — `block_device_operations`, `request_queue.queuedata`, and `gendisk` text RX-only; vtables constified after `add_disk`.
- **PAX_RANDKSTACK** — `ioctl` path is a syscall; per-syscall kernel-stack randomization re-applied before entering `blkdev_ioctl`.
- **PAX_REFCOUNT** — `request_queue.usage_counter`, `gendisk.open_partitions`, `block_device.bd_openers` saturating; teardown ordering enforced.
- **PAX_MEMORY_SANITIZE** — freed `request_queue`, `gendisk`, `block_device` objects zeroed; especially `bd_inode`/`bd_holder` to prevent UAF on concurrent open.
- **PAX_UDEREF** — STAC/CLAC bracket ioctl user-copies; `queue_lock` held only with IRQs off so user-pointer faults cannot reschedule into uaccess context.
- **PAX_RAP / kCFI** — `block_device_operations.{open, release, ioctl, compat_ioctl, revalidate_disk, getgeo}` RAP-signed.
- **PAX_AUTOSLAB** — per-driver request_queue + gendisk slab caches per-type-tagged.
- **PAX_SIZE_OVERFLOW** — sector arithmetic uses checked operators; offset math saturating.
- **GRKERNSEC_HIDESYM** — `/proc/partitions`, `/sys/block/*` redact queue + gendisk kaddrs for non-CAP_SYSLOG; only `dmesg_restrict`-cleared readers see raw pointers.
- **GRKERNSEC_DMESG** — `add_disk` / `del_gendisk` / queue-freeze prints rate-limited and gated by `dmesg_restrict`.
- **GRKERNSEC_IO** — raw `BLKRRPART`, `BLKFLSBUF`, `BLKDISCARD`, `BLKSECDISCARD` gated by CAP_SYS_ADMIN + gr-rbac role; chrooted tasks rejected unless explicitly delegated.
- **GRKERNSEC_CHROOT** — block-device opens (`/dev/sd*`, `/dev/nvme*`) inside chroot rejected unless gr-rbac role allows.

Per-doc rationale: `request_queue` + `gendisk` + `block_device` are the long-lived I/O lifecycle objects with the most aggressive concurrent-teardown patterns in the block layer; REFCOUNT-saturation on `usage_counter` / `open_partitions`, SANITIZE on freed queues/disks, USERCOPY on ioctl entrypoints, and GRKERNSEC_IO on raw partition-rescan / discard close the queue-teardown UAF and partition-rescan privilege-escalation paths.

## Open Questions

(none — request_queue + gendisk + bdev semantics fully specified by upstream)

## Out of Scope

- Per-driver block driver (cross-ref `drivers/00-overview.md`)
- 32-bit-only paths
- Implementation code
