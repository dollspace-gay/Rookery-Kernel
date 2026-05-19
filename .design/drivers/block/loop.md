# Tier-3: drivers/block/loop.c — loopback block device (file-backed, AIO/DIO, blk-mq, blk-cgroup)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/block/00-overview.md
upstream-paths:
  - drivers/block/loop.c
  - include/uapi/linux/loop.h
-->

## Summary

The loop driver maps a regular file (or another block device) into a `/dev/loopN` block device, letting userspace mount disk images, encrypted containers, install ISOs, build initrds, and provide writable overlays for containers (snapd, LXC, systemd-nspawn). Userland configures a loop via the `/dev/loop-control` misc-char node (`LOOP_CTL_{ADD,REMOVE,GET_FREE}`) and per-device ioctls (`LOOP_SET_FD`, `LOOP_CONFIGURE`, `LOOP_SET_STATUS64`, `LOOP_SET_DIRECT_IO`, `LOOP_SET_BLOCK_SIZE`, etc.). Internally each `loop_device` owns a `blk_mq_tag_set` driven by `loop_mq_ops`, dispatches each request to a per-cgroup workqueue (`worker_tree` rbtree), and performs the actual file IO via `vfs_iter_read`/`vfs_iter_write` or the AIO interface (`call_read_iter`/`call_write_iter` with `iocb->ki_flags |= IOCB_DIRECT`) when LO_FLAGS_DIRECT_IO is set. Partition scan, autoclear, read-only, and direct-IO modes are configurable via `loop_info64::lo_flags`. Encrypted-loop ciphers in the UAPI (`LO_CRYPT_*`) are obsolete and ignored — encryption now lives in dm-crypt.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct loop_device` | per-loop state (backing file, queue, tag set, workqueue, worker rbtree) | `drivers::block::loop::Loop` |
| `struct loop_cmd` | per-request driver state (placed in `request::pdu`) | `drivers::block::loop::LoopCmd` |
| `struct loop_worker` | per-blkcg worker on the rbtree | `drivers::block::loop::Worker` |
| `lo_fops` (`block_device_operations`) | per-loop fops: `lo_open`/`lo_release`/`lo_ioctl`/`lo_compat_ioctl`/`free_disk` | `drivers::block::loop::FOPS` |
| `loop_mq_ops` (`blk_mq_ops`) | `loop_queue_rq` + `lo_complete_rq` | `drivers::block::loop::MQ_OPS` |
| `loop_ctl_fops` + `loop_misc` | `/dev/loop-control` misc-char entry | `drivers::block::loop::CtlFOPS` |
| `LOOP_CONFIGURE` (0x4C0A) | atomic add + bind + configure | `LoopCtl::configure` |
| `LOOP_SET_FD` (0x4C00) / `LOOP_CLR_FD` (0x4C01) | bind backing FD / release | `LoopCtl::set_fd` / `_clr_fd` |
| `LOOP_SET_STATUS64` (0x4C04) / `LOOP_GET_STATUS64` (0x4C05) | settable flags + name + offset/sizelimit | `LoopCtl::set_status64` / `_get_status64` |
| `LOOP_CHANGE_FD` (0x4C06) | atomically swap RO backing file (e.g. snap refresh) | `LoopCtl::change_fd` |
| `LOOP_SET_CAPACITY` (0x4C07) | re-read size of backing file | `LoopCtl::set_capacity` |
| `LOOP_SET_DIRECT_IO` (0x4C08) | toggle LO_FLAGS_DIRECT_IO at runtime | `LoopCtl::set_direct_io` |
| `LOOP_SET_BLOCK_SIZE` (0x4C09) | set logical block size | `LoopCtl::set_block_size` |
| `LOOP_CTL_ADD` (0x4C80) / `LOOP_CTL_REMOVE` (0x4C81) / `LOOP_CTL_GET_FREE` (0x4C82) | `/dev/loop-control` ops | `LoopMisc::add` / `_remove` / `_get_free` |
| `loop_global_lock_killable` / `loop_global_unlock` | nested `loop_validate_mutex` + per-dev `lo_mutex` | `Loop::global_lock` / `_unlock` |
| `loop_configure` / `__loop_clr_fd` / `loop_set_status` | top-level config functions | `Loop::configure` / `_clr_fd` / `_set_status` |
| `loop_queue_rq` / `loop_handle_cmd` / `loop_process_work` | data-path | `Loop::queue_rq` / `_handle_cmd` / `_process_work` |
| `do_req_filebacked` / `lo_rw_aio` / `lo_write_simple` / `lo_read_simple` | per-op file IO | `Loop::do_req_filebacked` etc. |
| `loop_attr_group` (sysfs: backing_file, offset, sizelimit, autoclear, partscan, dio) | per-disk sysfs entries | `Loop::Attrs` |
| `loop_index_idr` + `loop_ctl_mutex` + `loop_validate_mutex` | global IDR + global locks | `LoopGlobal` |

## Compatibility contract

REQ-1: `/dev/loop-control` is a misc-char node at `LOOP_CTRL_MINOR` (237) registered via `misc_register(&loop_misc)`; ioctls `LOOP_CTL_ADD`, `LOOP_CTL_REMOVE`, `LOOP_CTL_GET_FREE` return new/freed loop index or -ENOSYS for unknown cmd.

REQ-2: `LOOP_MAJOR` (7) registered via `__register_blkdev(LOOP_MAJOR, "loop", loop_probe)`; per-loop minors derived from `part_shift = fls(max_part)` and `i << part_shift`.

REQ-3: Per-loop ioctl namespace 0x4C00..0x4C09 plus 0x4C0A (LOOP_CONFIGURE); compat ioctl handler (`lo_compat_ioctl`) accepts `LOOP_SET_STATUS` and `LOOP_GET_STATUS` via the 32-bit `loop_info` (old) translated up to `loop_info64`.

REQ-4: `LOOP_CONFIGURE` accepts only `LOOP_CONFIGURE_SETTABLE_FLAGS = LO_FLAGS_READ_ONLY | LO_FLAGS_AUTOCLEAR | LO_FLAGS_PARTSCAN | LO_FLAGS_DIRECT_IO`; all other bits in `loop_config::info::lo_flags` are rejected with -EINVAL.

REQ-5: `LOOP_SET_STATUS64` may only set/clear flags listed in `LOOP_SET_STATUS_SETTABLE_FLAGS` (AUTOCLEAR | PARTSCAN) and `LOOP_SET_STATUS_CLEARABLE_FLAGS` (AUTOCLEAR); other bits are EINVAL.

REQ-6: Backing-file validation (`loop_validate_file`) refuses cycles: if the new backing FD is itself a loop device, walk its chain via `is_loop_device` + `loop_validate_mutex` to detect cycles → -EBADSLT.

REQ-7: Direct-IO enabled only when backing file `f_mode & FMODE_CAN_ODIRECT`, the loop's logical block size ≥ backing device's logical block size, and `lo_offset` is aligned to `lo_min_dio_size`; otherwise silently demoted to buffered IO and the effective bit is reported in `loop_get_status`.

REQ-8: Per-loop blk-mq tag set: `nr_hw_queues = 1`, `queue_depth = hw_queue_depth` (default 128, module-param tunable at load time), `cmd_size = sizeof(struct loop_cmd)`, `flags = BLK_MQ_F_STACKING | BLK_MQ_F_BLOCKING`.

REQ-9: Per-loop worker isolation via per-blkcg `loop_worker` keyed by `blkcg_css` in `worker_tree` rbtree; idle workers expire after `LOOP_IDLE_WORKER_TIMEOUT` (60 s).

REQ-10: Pre-create `max_loop` (config `BLK_DEV_LOOP_MIN_COUNT`, runtime tunable via `loop.max_loop=N` only at load) loop devices; further devices created on-demand via `/dev/loop-control` or by accessing dead `/dev/loopN`.

REQ-11: Sysfs at `/sys/block/loopN/loop/{backing_file, offset, sizelimit, autoclear, partscan, dio}` — read-only LOOP_ATTR_RO entries.

REQ-12: Encrypted-loop ciphers (`LO_CRYPT_*`) in the UAPI header are deprecated; kernel ignores `lo_encrypt_type`/`lo_encrypt_key` fields and userland must use dm-crypt for encryption.

## Acceptance Criteria

- [ ] AC-1: `losetup -f --show /path/to/img` creates `/dev/loopN`; `losetup -d /dev/loopN` releases it.
- [ ] AC-2: `mkfs.ext4 /dev/loopN && mount /dev/loopN /mnt && fsck.ext4 /dev/loopN` cycle works.
- [ ] AC-3: `LOOP_SET_DIRECT_IO` toggles `O_DIRECT` IO path and `/sys/block/loopN/loop/dio` reflects the effective bit.
- [ ] AC-4: `LOOP_SET_STATUS64` partition-scan flag triggers `partscan` on the next open.
- [ ] AC-5: Cycle detection: `losetup /dev/loop1` then `LOOP_SET_FD(loop2, /dev/loop1)` then `LOOP_SET_FD(loop1, /dev/loop2)` returns -EBADSLT.
- [ ] AC-6: `blktests` block/loop subset passes.
- [ ] AC-7: blk-cgroup proxy: a cgroup-throttled writer through a loop-backed mount honors the throttle (verifiable via `blkio.throttle.io_service_bytes`).

## Architecture

Each `loop_device` owns a `gendisk`, a `request_queue`, a `blk_mq_tag_set`, and a `workqueue_struct` plus a `worker_tree` rbtree keyed by `blkcg_css`. Requests arriving via `loop_queue_rq` are inserted into either the per-blkcg worker's `cmd_list` (when CONFIG_BLK_CGROUP and the request has a `bio_blkcg_css`) or the `rootcg_cmd_list` otherwise, then `queue_work` dispatches the worker. The worker function (`loop_workfn` / `loop_rootcg_workfn`) calls `loop_process_work` which iterates the list and runs `loop_handle_cmd` for each.

`loop_handle_cmd` re-associates the executing kthread with the captured `blkcg_css` and `memcg_css` (via `kthread_associate_blkcg` and `set_active_memcg`), then calls `do_req_filebacked` which dispatches per `req_op(rq)` to one of: `lo_read_simple` / `lo_write_simple` (buffered, using `vfs_iter_read`/`_write`), `lo_rw_aio` (AIO path using `call_read_iter`/`_write_iter` with `IOCB_DIRECT`), `lo_discard` (`vfs_fallocate` with `FALLOC_FL_PUNCH_HOLE`), `lo_req_flush` (`vfs_fsync`), or `lo_fallocate` (write-zeroes via `FALLOC_FL_ZERO_RANGE`). AIO completion is delivered via `lo_rw_aio_complete` → `blk_mq_complete_request`.

Lock hierarchy: `loop_validate_mutex` (global, taken on cycle-check paths) → `lo_mutex` (per-device, taken on configure/clr_fd/set_status) → `lo_lock` spinlock (per-device, taken on state transitions). The `loop_ctl_mutex` is a separate global mutex protecting `loop_index_idr` add/remove. Workqueue submission uses `lo_work_lock` spinlock to protect `worker_tree` and `rootcg_cmd_list`.

Lifetime: `gendisk` is freed via the block layer's `free_disk` callback (`lo_free_disk`), which kfrees the `loop_device`. State machine: `Lo_unbound` (no backing file) → `Lo_bound` (configured) → `Lo_rundown` (clr_fd in flight, no new IO) → back to `Lo_unbound`, or `Lo_deleting` (removal in flight, then freed).

## Hardening

- `loop_configure` and `LOOP_SET_FD` validate the backing FD's superblock — refuses to bind a backing file from the same `struct super_block` recursively via `loop_validate_file`.
- `lo_offset + lo_sizelimit` overflow checked against `loff_t` before use; computed loop size shifted right 9 to fit in `sector_t`.
- `partscan` requires the loop to be in `Lo_unbound` or the bit must be cleared first; refuses to flip it on a mounted device.
- Direct-IO promotion checks `f_mode & FMODE_CAN_ODIRECT` AND alignment AND backing-device block size — silently demotes on mismatch.
- AIO completion path uses `atomic_t ref` on `loop_cmd` to coordinate multi-segment AIO; refcount-overflow trapped.
- Per-blkcg worker rbtree bounded by `LOOP_IDLE_WORKER_TIMEOUT` reaper (60 s); defense against blkcg-storm DoS.
- `LOOP_CHANGE_FD` requires backing files to be read-only and same size; refuses on RW or size-mismatch.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `struct loop_info64`, `struct loop_config`, and compat `struct loop_info` in `lo_compat_ioctl`; refuse copy_from_user/copy_to_user that touches any other slab.
- **PAX_KERNEXEC** — `lo_fops`, `loop_mq_ops`, `loop_ctl_fops`, `loop_misc`, and the per-attribute show callbacks live in `__ro_after_init` kernel text.
- **PAX_RANDKSTACK** — randomize kernel stack offset across `lo_ioctl`, `loop_control_ioctl`, `loop_configure`, `loop_queue_rq`, and `loop_handle_cmd` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `loop_cmd::ref` (AIO multi-segment completion), `gendisk` holder count, backing-file `struct file`'s refcount.
- **PAX_MEMORY_SANITIZE** — zero-on-free for the legacy `lo_encrypt_key` field of `loop_info64`, all `loop_cmd` slabs, `bio_vec` arrays in `cmd->bvec`, and worker-tree entries so backing-file paths and cgroup css pointers do not bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every `lo_ioctl`, `lo_compat_ioctl`, and `loop_control_ioctl` entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `lo_fops` (`.open`, `.release`, `.ioctl`, `.compat_ioctl`, `.free_disk`) and `loop_mq_ops` (`.queue_rq`, `.complete`) marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-loop `loop_device` pointer disclosure behind CAP_SYSLOG; suppress `%p` for backing-file inode pointers in tracepoints.
- **GRKERNSEC_DMESG** — restrict `loop_validate_file` cycle-rejection banners and ENOMEM banners to CAP_SYSLOG so attackers cannot probe backing-store topology via dmesg.
- **Backing-file FD CAP_SYS_ADMIN gating** — `LOOP_SET_FD`, `LOOP_CONFIGURE`, `LOOP_CHANGE_FD` require CAP_SYS_ADMIN in the user namespace owning the loop device; refuse to bind a backing FD owned by a different user namespace than the caller.
- **PAX_USERCOPY on `LOOP_SET_FD` ioctl path** — `loop_config` and `loop_info64` slab caches whitelisted; the legacy `lo_encrypt_key` byte array (32 bytes) is zeroed on copy-out and never leaked from kernel slab.
- **MEMORY_SANITIZE on crypto-key fields** — even though the in-kernel crypto path is deprecated, the `lo_encrypt_key` field is zeroed-on-free on every `__loop_clr_fd` and `loop_remove` path so historical key bytes never bleed across loop-device reuse.
- **Cycle-check enforcement** — `loop_validate_file` walks the loop chain under `loop_validate_mutex` and rejects with -EBADSLT; cycle-rejection counted in a hardening counter and rate-limited from dmesg.
- **Partscan privilege boundary** — toggling LO_FLAGS_PARTSCAN requires CAP_SYS_ADMIN AND `Lo_unbound` state OR a freshly-configured device with no openers other than the configurer.
- **Workqueue concurrency cap** — per-loop workqueue created with `WQ_UNBOUND | WQ_FREEZABLE` + bounded `max_active`; per-blkcg worker rbtree pruned by `LOOP_IDLE_WORKER_TIMEOUT` reaper.

Rationale: loop is the canonical "trusted blob from userspace becomes a kernel-managed block device" boundary. A relaxed cycle check, an unsanitized encryption-key field, a writable `lo_fops`, an unbounded worker rbtree, or a missed user-namespace check on the backing FD turns a user-controllable file descriptor into a kernel-controlled storage stack with cgroup-credential and memcg authority. RAP/kCFI on every ops table, CAP_SYS_ADMIN on every config plane, memory-sanitize on key/path slabs, refcount-overflow trap on AIO ref, and cycle detection under a global mutex turn the loop driver from "I/O virtualisation toy" into a structural enforcement boundary.

## Open Questions

- [ ] Q1: Should the kernel reject FDs that span container user-namespace boundaries by default, or continue to delegate to capability checks only?
- [ ] Q2: Is removing the deprecated `lo_encrypt_*` UAPI fields ABI-safe in a Rookery 1.0 release, or should they remain for compat with very old `losetup` binaries?
- [ ] Q3: Should the per-blkcg worker rbtree be replaced with a per-CPU `workqueue` + cgroup tag (lower memory, less precise) given the historical motivation was throughput per-cgroup, which blk-cgroup throttling now handles directly?

## Verification

- `losetup -a` enumerates active loops with backing-file + offset + ro/rw + autoclear flags.
- `cat /sys/block/loopN/loop/{backing_file,offset,sizelimit,autoclear,partscan,dio}`.
- `dmesg | grep -i loop` after `LOOP_SET_FD`, `LOOP_CLR_FD`, and cycle-detection rejections.
- `blktests` block/008, block/009 (loop-specific subsets), and `xfstests` generic with loop-backed image.
- `losetup --direct-io=on /dev/loopN` flips `/sys/block/loopN/loop/dio` to `1`.
- ftrace `block:block_rq_issue` and `block:block_rq_complete` against the loop while running fio.
