# Tier-3: drivers/block/null_blk/* — null block device for blk-mq perf testing (zoned, memory-backed, badblocks, fault injection)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/block/00-overview.md
upstream-paths:
  - drivers/block/null_blk/main.c
  - drivers/block/null_blk/null_blk.h
  - drivers/block/null_blk/zoned.c
  - drivers/block/null_blk/trace.h
  - drivers/block/null_blk/trace.c
-->

## Summary

`null_blk` is the canonical blk-mq performance-testing and feature-modelling driver: it produces a block device with no backing store (responses are discarded or stored in an in-memory radix tree) so the block layer, scheduler, and filesystem code paths above it can be benchmarked without storage hardware. Configuration is via configfs (`/sys/kernel/config/nullb/`), module parameters, and a small ioctl surface inherited from the generic block layer. It models real-world device features end-to-end: zoned block devices (`null_blk/zoned.c`, host-managed SMR semantics), discard, write-zeroes, FUA, FUA, memory-backed (radix-tree-stored writes for correctness tests), write-back cache emulation, bandwidth throttling, badblocks emulation, and fault injection (timeout, requeue, init_hctx). It is the reference target for `blktests` and for new blk-mq features.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nullb_device` | per-device config + state (configfs `config_group`, flags, radix trees, zones, fault attrs) | `drivers::block::null_blk::Device` |
| `struct nullb` | per-instance runtime state (gendisk, tag set, queues, hrtimers) | `drivers::block::null_blk::Instance` |
| `struct nullb_queue` | per-queue state (poll list, requeue selection) | `drivers::block::null_blk::Queue` |
| `struct nullb_cmd` | per-request state (in `request::pdu`) | `drivers::block::null_blk::Cmd` |
| `struct nullb_page` | per-page memory-backed slot in radix tree | `drivers::block::null_blk::Page` |
| `struct nullb_zone` | per-zone state for zoned devices | `drivers::block::null_blk::Zone` |
| `null_ops` (`block_device_operations`) | `null_open`/`null_release`/`null_report_zones` | `drivers::block::null_blk::FOPS` |
| `null_mq_ops` (`blk_mq_ops`) | `null_queue_rq`, `null_queue_rqs`, `null_complete_rq`, `null_init_hctx`, `null_timeout_rq`, `null_poll` | `drivers::block::null_blk::MQ_OPS` |
| `nullb_subsys` (`configfs_subsystem`) | configfs root `/sys/kernel/config/nullb/` | `drivers::block::null_blk::ConfigfsRoot` |
| `nullb_device_attrs` + `NULLB_DEVICE_ATTR_*` | per-attribute configfs entries (size, blocksize, queue_mode, irqmode, hw_queue_depth, zoned, zone_size, zone_capacity, mbps, cache_size, discard, memory_backed, ...) | `drivers::block::null_blk::Attrs` |
| `null_alloc_dev` / `null_free_dev` / `null_add_dev` / `null_del_dev` | per-device lifecycle | `Device::alloc` / `_free` / `Instance::add` / `_del` |
| `null_handle_cmd` / `null_process_cmd` | per-request dispatch | `Instance::handle_cmd` / `_process_cmd` |
| `null_init_zoned_dev` / `null_register_zoned_dev` / `null_process_zoned_cmd` / `null_report_zones` | zoned subsystem | `Zoned::init` / `_register` / `_process_cmd` / `_report_zones` |
| `null_handle_memory_backed` / `null_handle_discard` / `null_handle_badblocks` | per-feature handlers | `MemBack::handle` / `Discard::handle` / `Badblocks::handle` |
| `null_timeout_attr` / `null_requeue_attr` / `null_init_hctx_attr` | fault-injection attrs (DECLARE_FAULT_ATTR) | `FaultInject::Attrs` |
| `g_*` module params (`submit_queues`, `poll_queues`, `home_node`, `irqmode`, `queue_mode`, `gb`, `bs`, `nr_devices`, `zoned`, `zone_size`, `mbps`, ...) | module-load defaults | `Module::Params` |

## Compatibility contract

REQ-1: Module loads with `modprobe null_blk` and pre-creates `nr_devices` devices (default 1); additional devices created on-demand via `mkdir /sys/kernel/config/nullb/<name>` (configfs).

REQ-2: configfs hierarchy `/sys/kernel/config/nullb/<name>/` exposes every `nullb_device` attribute as a file; attribute changes after `power=1` (configured) are refused with -EBUSY.

REQ-3: Per-device blk-mq tag set: `nr_hw_queues = submit_queues + poll_queues`, `queue_depth = hw_queue_depth` (default 64), `cmd_size = sizeof(struct nullb_cmd)`, `flags` per `g_blocking` and `g_shared_tag_bitmap`.

REQ-4: Queue modes: `NULL_Q_BIO` (legacy bio path, deprecated), `NULL_Q_RQ` (legacy request-based, deprecated), `NULL_Q_MQ` (default, blk-mq); the historical `queue_mode` module param keeps the enum.

REQ-5: IRQ modes: `NULL_IRQ_NONE` (sync complete in `queue_rq`), `NULL_IRQ_SOFTIRQ` (defer to `blk_mq_complete_request`), `NULL_IRQ_TIMER` (hrtimer-completed for latency-injection).

REQ-6: Zoned support (CONFIG_BLK_DEV_ZONED): host-managed model with conventional + sequential-write-required zones; per-zone state machine via `nullb_zone::cond`; `null_report_zones` callback exposes zone descriptors per `blk_zone` ABI.

REQ-7: Memory-backed mode (`memory_backed=1`): writes stored in `nullb_device::data` radix tree, reads serve from radix tree (or zeros for unmapped sectors); supports cache emulation via `nullb_device::cache` radix tree.

REQ-8: Discard support (`discard=1`): `null_handle_discard` walks the radix tree freeing per-page `nullb_page`s in `[sector, sector+nr_sectors)`.

REQ-9: Badblocks emulation (`badblocks` configfs file): `nullb_device::badblocks` (struct `badblocks`) tracks ranges; reads/writes hitting a badblock return `BLK_STS_IOERR` (or partial-IO IOERR with `badblocks_partial_io=1`).

REQ-10: Fault injection (CONFIG_BLK_DEV_NULL_BLK_FAULT_INJECTION): `null_timeout_attr`, `null_requeue_attr`, `null_init_hctx_attr` driven by `g_timeout_str`, `g_requeue_str`, `g_init_hctx_str` module params (format `<interval>,<probability>,<space>,<times>`).

REQ-11: Throttling (`mbps=N`): per-`nullb` `bw_timer` hrtimer caps throughput; resets `cur_bytes` every `TIMER_INTERVAL` tick.

REQ-12: Trace events from `null_blk/trace.h`: `nullb_zone_op`, `nullb_report_zones` — gated by `tracing_subsys` permissions.

## Acceptance Criteria

- [ ] AC-1: `modprobe null_blk` creates `/dev/nullb0`; `lsblk -d /dev/nullb0` reports the configured size.
- [ ] AC-2: `mkdir /sys/kernel/config/nullb/test1 && echo 0 > /sys/kernel/config/nullb/test1/zoned && echo 1024 > /sys/kernel/config/nullb/test1/size && echo 1 > /sys/kernel/config/nullb/test1/power` creates a second device.
- [ ] AC-3: `fio --rw=randread --ioengine=libaio --bs=4k --iodepth=128 --filename=/dev/nullb0` saturates the configured queue depth and produces zero-byte responses.
- [ ] AC-4: Zoned: `modprobe null_blk zoned=1 zone_size=64` then `blkzone report /dev/nullb0` shows the zone descriptor list; `blkzone reset` transitions zones to EMPTY.
- [ ] AC-5: Memory-backed: `dd if=/dev/urandom of=/dev/nullb0 bs=4k count=1024 && dd if=/dev/nullb0 of=/tmp/out bs=4k count=1024 && cmp` round-trips correctly.
- [ ] AC-6: Fault injection: `null_timeout_str=1,100,0,1` triggers exactly one timeout on the first request; `blk_mq_timeout_handler` rerun observed in ftrace.
- [ ] AC-7: `blktests` block/null_blk subset passes.

## Architecture

Each `nullb_device` is a configfs `config_group` owning a forest of per-attribute callbacks; mkdir under `/sys/kernel/config/nullb/` calls `null_alloc_dev` which allocates the group, populates default attrs (from `g_*` module-param defaults), and adds the empty device to `nullb_list`. Writing `power=1` invokes `null_add_dev` which allocates the `nullb` instance, sets up the `blk_mq_tag_set` (or attaches to the global shared `tag_set` if `shared_tags=1`), allocates per-queue arrays, sets up zoned/memory-backed/cache subsystems, configures `queue_limits`, and finally calls `add_disk`.

Data path: `null_queue_rq` (or batched `null_queue_rqs`) → `null_handle_cmd` examines `nullb_device` flags and routes through `null_process_zoned_cmd` (zoned), `null_handle_memory_backed` (memory-backed), `null_handle_discard` (discards), `null_handle_badblocks` (badblock simulation), or simply discards the data (default null behaviour). Completion is per IRQ mode: NULL_IRQ_NONE completes synchronously in `queue_rq`, NULL_IRQ_SOFTIRQ defers via `blk_mq_complete_request`, NULL_IRQ_TIMER arms a per-cmd hrtimer for latency injection.

Lock hierarchy: global `lock` mutex (around `nullb_list` and configfs subsys) → per-device `nullb_device::zone_res_lock` spinlock (zoned resource management: max_open_zones, max_active_zones) → per-zone `nullb_zone::spinlock` or `nullb_zone::mutex` (memory-backed uses the mutex variant because writes may sleep on radix-tree alloc). Per-queue spinlocks (`nullb_queue::poll_lock`) protect the poll list.

Lifetime: `null_del_dev` triggered by `rmdir /sys/kernel/config/nullb/<name>` or by `power=0` write: `del_gendisk` → `blk_mq_free_tag_set` (if not shared) → free zones → free radix trees → free `nullb_device`. Forced module unload (`rmmod null_blk`) iterates `nullb_list` and removes every device.

## Hardening

- configfs attribute parsing uses kstrtoul/kstrtobool with explicit range checks; refuses negative or out-of-range values.
- `zone_size`, `zone_capacity`, `zone_nr_conv`, `zone_max_open`, `zone_max_active` validated against `disk size / zone_size` at `null_add_dev` time.
- `null_alloc_dev` zero-initialises the `nullb_device` slab so stale fault-injection state cannot bleed across reuse.
- Module-param `nr_devices` capped at module-load; runtime device count bounded by configfs slab budget.
- Fault-injection attrs gated by `CONFIG_BLK_DEV_NULL_BLK_FAULT_INJECTION` — completely absent from the binary when disabled.
- `null_handle_discard` validates `[sector, sector+nr_sectors)` against disksize before walking the radix tree.
- Per-zone state transitions checked against `blk_zone_cond` enum; invalid transitions return `BLK_STS_ZONE_INVALID_CMD`.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist configfs attribute scratch buffers, `blk_zone` descriptor arrays returned through `null_report_zones`, and fault-injection `g_*_str` static buffers; refuse copy_to_user from any other slab.
- **PAX_KERNEXEC** — `null_ops`, `null_mq_ops`, `nullb_subsys`, configfs item_type/group_type tables placed in `__ro_after_init` kernel text.
- **PAX_RANDKSTACK** — randomize kernel stack across `null_queue_rq`, `null_init_hctx`, `null_handle_cmd`, `null_process_zoned_cmd`, `null_report_zones`, and every configfs `_store` callback entry.
- **PAX_REFCOUNT** — saturating `refcount_t` on configfs `config_item::ci_kref`, `gendisk` holder count, and per-device `nullb` references.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `nullb_page` slabs, radix-tree internal nodes, `nullb_zone` arrays, and `nullb_device` slabs so memory-backed data and zone state do not bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every configfs `_store` and ioctl entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `null_ops` (`.open`, `.release`, `.report_zones`), `null_mq_ops` (`.queue_rq`, `.queue_rqs`, `.complete`, `.init_hctx`, `.timeout`, `.poll`), and every configfs `_store`/`_show` callback marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-device pointer disclosure (e.g. `nullb_device`, `nullb` slab addresses) behind CAP_SYSLOG; suppress `%p` for radix-tree node pointers in tracepoints.
- **GRKERNSEC_DMESG** — restrict zone-state-violation, badblock-hit, and fault-injection banners to CAP_SYSLOG so attackers cannot map injection topology via dmesg.
- **CAP_SYS_ADMIN on configfs mutation** — `mkdir`/`rmdir` and every attribute `_store` callback under `/sys/kernel/config/nullb/` require CAP_SYS_ADMIN in the user namespace owning configfs; refuse unprivileged writers.
- **Zone-config PAX_USERCOPY** — `null_report_zones` populates a kernel-allocated `blk_zone[]` then copies out through `blk_report_zones` core path with bounded length; refuses `nr_zones * sizeof(struct blk_zone)` larger than `INT_MAX / sizeof(struct blk_zone)`.
- **Tiny ioctl surface** — the driver inherits only the generic block-layer ioctls (BLKGETSIZE, BLKDISCARD, etc.); no per-driver ioctl handler is registered, eliminating an entire attack surface.
- **blktrace gate** — `nullb_zone_op` and `nullb_report_zones` tracepoints require CAP_SYS_ADMIN on `tracefs` registration; defense against per-IO state disclosure via tracefs.
- **Fault-injection gated** — `null_timeout_attr` / `null_requeue_attr` / `null_init_hctx_attr` only present when `CONFIG_BLK_DEV_NULL_BLK_FAULT_INJECTION=y`; debugfs entries (`fail_*_real`) require CAP_SYS_ADMIN.
- **Module-param immutability** — every `g_*` module param exposed `0444`; refuse runtime retuning. Per-device retuning happens only via configfs while `power=0`.
- **Per-zone state-machine invariants** — `null_process_zoned_cmd` enforces conventional vs. sequential semantics; rejects writes that would cross conventional/sequential boundary.

Rationale: null-blk is small but it is the kernel's reference target for the entire blk-mq feature surface — every new block-layer feature is exercised through it first. A relaxed configfs attribute, a writable `null_mq_ops`, an unchecked `nr_zones` in `report_zones`, or a leaked memory-backed page turns a benchmarking toy into an arbitrary kernel-memory write/read primitive (memory-backed mode allocates pages from kernel slab and stores user data). RAP/kCFI on every vtable, CAP_SYS_ADMIN on every configfs mutation, MEMORY_SANITIZE on the per-page slab, fault-injection compile-time gating, and SIZE_OVERFLOW on `nr_zones` turn null-blk into a hardened reference implementation rather than a footgun.

## Open Questions

- [ ] Q1: Should memory-backed mode be restricted to a dedicated mempool with explicit upper bound, given a configured `size=1024G` device would otherwise allow allocating up to 1 TB of kernel slab on writes?
- [ ] Q2: Is the historical `NULL_Q_BIO`/`NULL_Q_RQ` queue mode worth maintaining, or should it be removed in Rookery 1.0 to shrink the attack surface?
- [ ] Q3: Should configfs attribute mutation be journaled (audit subsystem hook) for tamper-evidence in production benchmarking environments?

## Verification

- `modprobe null_blk submit_queues=4 poll_queues=2 zoned=0` and `cat /sys/block/nullb0/queue/nr_hw_queues` returns 6.
- `mkdir /sys/kernel/config/nullb/zoned1 && echo 1 > /sys/kernel/config/nullb/zoned1/zoned && echo 1 > /sys/kernel/config/nullb/zoned1/power`; verify `/dev/nullb1` is zoned via `blkzone report`.
- `fio --rw=randwrite --iodepth=128 --bs=4k --runtime=10 --filename=/dev/nullb0` reports configured throughput.
- `blktests` block/null_blk subset passes (block/004, block/005, block/008, block/024 with null_blk).
- ftrace `block:*` events flow through the driver under load.
- `cat /sys/kernel/config/nullb/<name>/*` enumerates all attributes; mutating any while `power=1` returns -EBUSY.
