# Tier-3: drivers/block/zram/* — compressed RAM block device (LZO/LZ4/LZ4HC/zstd/deflate/842, swap, multi-comp, writeback)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/block/00-overview.md
upstream-paths:
  - drivers/block/zram/zram_drv.c
  - drivers/block/zram/zram_drv.h
  - drivers/block/zram/zcomp.c
  - drivers/block/zram/zcomp.h
  - drivers/block/zram/backend_lzo.c
  - drivers/block/zram/backend_lzorle.c
  - drivers/block/zram/backend_lz4.c
  - drivers/block/zram/backend_lz4hc.c
  - drivers/block/zram/backend_zstd.c
  - drivers/block/zram/backend_deflate.c
  - drivers/block/zram/backend_842.c
-->

## Summary

`zram` is a compressed RAM-backed block device used primarily as compressed swap (so a system can swap into RAM rather than to disk) and as a backing for tmpfs-style ephemeral storage in resource-constrained environments (Chrome OS, Android, embedded distributions). Each `/dev/zramN` is configured by writing a `disksize` to sysfs; subsequent writes are compressed via a configurable algorithm (LZO-RLE, LZO, LZ4, LZ4HC, zstd, deflate, 842) and stored in `zsmalloc` slabs; reads decompress on-the-fly. Optional features include same-page filling (zero/repeated pages stored as a single value), huge-page tracking (incompressible pages stored uncompressed), backing-device writeback (offload to disk when memory pressure builds), multi-compression (CONFIG_ZRAM_MULTI_COMP, up to 4 priorities, with `recompress` for opportunistic re-pack), idle tracking, and entry-time accounting (CONFIG_ZRAM_TRACK_ENTRY_ACTIME).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct zram` | per-zram state (table, mem_pool, comps[], disk, dev_lock, stats, disksize, comp_algs[], backing_dev, bitmap) | `drivers::block::zram::Zram` |
| `struct zram_table_entry` | per-slot (per-disk-page) entry (handle + flags) | `drivers::block::zram::Slot` |
| `struct zcomp` | per-algorithm compression state (per-CPU streams, ops vtable) | `drivers::block::zram::Zcomp` |
| `struct zcomp_ops` | per-algorithm vtable (`create_ctx`, `destroy_ctx`, `compress`, `decompress`, `setup_params`, `release_params`) | `drivers::block::zram::ZcompOps` |
| `struct zcomp_strm` | per-CPU per-zcomp stream (buffer, local_copy, ctx) | `drivers::block::zram::ZcompStrm` |
| `struct zram_stats` | per-zram atomic counters (compr_data_size, failed_reads/writes, notify_free, same_pages, huge_pages, pages_stored, max_used_pages, miss_free, bd_*) | `drivers::block::zram::Stats` |
| `zram_devops` (`block_device_operations`) | `zram_open`/`zram_release`/`zram_submit_bio`/`zram_rw_page` (where applicable) | `drivers::block::zram::FOPS` |
| `disksize_store` (sysfs `disksize`) | one-shot init: alloc zsmalloc pool + comp + table | `Zram::set_disksize` |
| `reset_store` (sysfs `reset`) | tear down, free table/pool, return to unconfigured | `Zram::reset` |
| `comp_algorithm_show/_store` (sysfs `comp_algorithm`) | per-priority algo selection | `Zram::set_comp_algorithm` |
| `recomp_algorithm_store` (sysfs `recomp_algorithm`) | secondary algo for `recompress` (CONFIG_ZRAM_MULTI_COMP) | `Zram::set_recomp_algorithm` |
| `backing_dev_store` (sysfs `backing_dev`) | bind a regular file (`filp_open`) as writeback target | `Zram::set_backing_dev` |
| `writeback_store` (sysfs `writeback`) | flush idle/huge/incompressible slots to backing dev | `Zram::writeback` |
| `mem_limit_store` / `mem_used_max_store` (sysfs) | per-zram memory caps | `Zram::set_mem_limit` |
| `idle_store` / `mark_idle` | mark slots IDLE for selective writeback | `Zram::mark_idle` |
| `mm_stat` / `bd_stat` / `io_stat` (sysfs read-only) | per-zram stats | `Zram::Stats` |
| `zcomp_create` / `zcomp_destroy` / `zcomp_compress` / `zcomp_decompress` | per-zcomp lifecycle + ops | `Zcomp::create` / `_destroy` / `_compress` / `_decompress` |
| `backend_{lzo, lzorle, lz4, lz4hc, zstd, deflate, 842}` (`zcomp_ops`) | per-algorithm backends | `Backends::*` |

## Compatibility contract

REQ-1: `ZRAM_MAJOR` (251 default) registered via `register_blkdev`; per-zram minor allocated via `idr_alloc(&zram_index_idr, ...)`.

REQ-2: `/dev/zram-control` misc-char node (`zram_control`) exposes `hot_add` and `hot_remove` attributes for creating/destroying zram devices at runtime; mirrors loop-control semantics.

REQ-3: One-shot configuration: writing `disksize` initialises the device and freezes algorithm choice; subsequent `disksize_store` returns -EBUSY until `reset_store` (which requires zero openers).

REQ-4: Per-slot metadata in `zram_table_entry`: `unsigned long handle` (zsmalloc handle, or special values) + per-slot flags (`ZRAM_SAME`, `ZRAM_ENTRY_LOCK`, `ZRAM_WB`, `ZRAM_PP_SLOT`, `ZRAM_HUGE`, `ZRAM_IDLE`, `ZRAM_INCOMPRESSIBLE`, `ZRAM_COMP_PRIORITY_BIT1/2`).

REQ-5: Compression algorithms compile-time gated by `CONFIG_ZRAM_BACKEND_{LZO,LZ4,LZ4HC,ZSTD,DEFLATE,842}`; runtime selection via `comp_algorithm` sysfs (default `default_compressor` kconfig).

REQ-6: Multi-comp (CONFIG_ZRAM_MULTI_COMP): up to `ZRAM_MAX_COMPS = 4` primary+secondary algorithms; per-priority via `comp_algorithm_store` (priority 0 only) and `recomp_algorithm_store` (priorities 1..3); `recompress` sysfs op re-packs idle/huge slots through secondary algorithms.

REQ-7: Same-page filling: pages consisting entirely of one repeated `unsigned long` value stored as that value in `handle` itself (no zsmalloc allocation), flagged `ZRAM_SAME`; defense against the all-zero swap-page case dominating zram footprint.

REQ-8: Backing-device writeback (CONFIG_ZRAM_WRITEBACK): `backing_dev` sysfs binds a regular file via `filp_open(O_RDWR|O_LARGEFILE|O_EXCL)` whose `i_size` provides the offload pool; `writeback` sysfs flushes selected slots (`idle`, `huge`, `incompressible`) into the backing device with per-slot bitmap tracking. `compressed_writeback` option writes compressed data instead of decompressed.

REQ-9: Per-slot bit-lock (`ZRAM_ENTRY_LOCK`) on `zram_table_entry::__lock` provides per-slot mutual exclusion; saves memory vs. a per-slot mutex.

REQ-10: Sysfs read-only stat groups: `mm_stat` (orig_data_size, compr_data_size, mem_used_total, mem_limit, mem_used_max, same_pages, pages_compacted, huge_pages, huge_pages_since), `bd_stat` (bd_count, bd_reads, bd_writes), `io_stat` (failed_reads, failed_writes, invalid_io, notify_free).

REQ-11: debugfs (`CONFIG_ZRAM_MEMORY_TRACKING`): per-zram `block_state` file iterates the entry table — gated by CAP_SYS_ADMIN.

REQ-12: PM/freeze: `dev_lock` (`rw_semaphore`) taken on every IO and on every sysfs mutator; ensures concurrent reset/disksize cannot race in-flight bio.

## Acceptance Criteria

- [ ] AC-1: `modprobe zram num_devices=4` creates `/dev/zram0..3`; `lsblk -d /dev/zram0` reports `0B` until `disksize` is written.
- [ ] AC-2: `echo 1G > /sys/block/zram0/disksize && mkswap /dev/zram0 && swapon /dev/zram0` enables compressed swap; `swapon -s` shows it.
- [ ] AC-3: `echo lz4 > /sys/block/zram0/comp_algorithm` before `disksize` set selects lz4 backend; after `disksize` set, write returns -EBUSY.
- [ ] AC-4: `cat /sys/block/zram0/mm_stat` shows `orig_data_size > compr_data_size` after writing compressible data.
- [ ] AC-5: Backing-dev writeback: `echo /path/to/file > /sys/block/zram0/backing_dev && echo idle > /sys/block/zram0/idle && echo idle > /sys/block/zram0/writeback`; `cat /sys/block/zram0/bd_stat` shows non-zero bd_writes.
- [ ] AC-6: `echo 1 > /sys/block/zram0/reset` (with zero openers) returns the device to unconfigured state.
- [ ] AC-7: Multi-comp: `echo "1 zstd" > /sys/block/zram0/recomp_algorithm` then `echo idle > /sys/block/zram0/recompress` re-packs idle slots through zstd.
- [ ] AC-8: `blktests` block/zram subset passes.

## Architecture

A fresh `zram` is empty: no `table`, no `mem_pool`, no per-zcomp state. Writing `disksize` triggers `disksize_store` which allocates `table[disksize / PAGE_SIZE]` (one entry per disk page), calls `zs_create_pool` to allocate the zsmalloc pool, and `zcomp_create` for each configured priority (creating per-CPU `zcomp_strm` streams via cpuhotplug callbacks).

Data path is bio-based (not blk-mq) for zram — `zram_submit_bio` is the per-bio entry; it iterates the bio's bio_vec list calling `zram_bvec_rw` per segment which dispatches to `zram_bvec_read` or `zram_bvec_write`. Write: copy the user page into `zstrm->local_copy`, attempt same-page detection (`page_same_filled`), otherwise call `zcomp_compress` → if compressed size ≥ PAGE_SIZE flag as ZRAM_HUGE and store raw (one zsmalloc handle per page), else allocate compressed-size handle via `zs_malloc(pool, compressed_size, GFP_NOIO | __GFP_HIGHMEM | __GFP_MOVABLE)` and `memcpy` into it. Read: if ZRAM_SAME, fill page with stored value; if ZRAM_HUGE, copy raw from handle; if ZRAM_WB, read from backing device at recorded bitmap offset; otherwise `zs_map_object` + `zcomp_decompress` → kunmap.

Lock hierarchy: `zram->dev_lock` (rw_semaphore, taken in shared mode on every IO and in exclusive mode on every config mutation — disksize, reset, comp_algorithm, backing_dev, writeback) → per-slot bit-lock on `ZRAM_ENTRY_LOCK` flag → zsmalloc internal locks (covered in `mm/zsmalloc.md`). Per-CPU `zcomp_strm` access is preempt-disabled via `local_lock`.

Lifetime: `zram_reset_device` (called from `reset_store` or `zram_remove`) requires `disk_openers(disk) == 0` (verified under `dev_lock` write); drains in-flight IO; iterates `table` freeing every non-empty slot; `zs_destroy_pool`; `zcomp_destroy` per priority; frees `table`; transitions back to unconfigured.

## Hardening

- `disksize_store` clamps `disksize` to a sane upper bound and refuses zero/non-page-multiple values.
- `comp_algorithm_store` and `recomp_algorithm_store` reject algorithms not compiled in (`lookup_backend_ops` returns NULL) with -EINVAL.
- `backing_dev_store` opens with `O_EXCL`; refuses to use a backing file already opened exclusively elsewhere; validates the path resolves within an acceptable filesystem (no procfs/sysfs/devtmpfs).
- `writeback_store` and `idle_store` take `dev_lock` in shared mode and require backing dev configured.
- `zram_remove` waits for `disk_openers == 0` under `disk->open_mutex`; refuses while any opener present.
- Compression failure (`zcomp_compress` returns non-zero) falls back to ZRAM_HUGE raw storage; never panics.
- Per-CPU `zcomp_strm::buffer` (2 pages from `vzalloc`) sized to handle worst-case expansion when compressed size exceeds original.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `mm_stat`, `bd_stat`, `io_stat` sysfs scratch buffers, `comp_algorithm` algorithm-name buffers, and `backing_dev` path buffers; refuse copy_to_user/copy_from_user that touches `zram_table_entry`, zsmalloc handles, or `zcomp_strm::buffer`.
- **PAX_KERNEXEC** — `zram_devops`, every sysfs `DEVICE_ATTR_*` `_show`/`_store` callback table, `backends[]` array, and per-backend `zcomp_ops` placed in `__ro_after_init` kernel text.
- **PAX_RANDKSTACK** — randomize kernel stack across `zram_submit_bio`, `zram_bvec_rw`, `zcomp_compress`, `zcomp_decompress`, every sysfs `_store` entry, and `writeback_store`.
- **PAX_REFCOUNT** — saturating `refcount_t` on `gendisk` holder count, `backing_dev` file refcount, and per-zcomp algorithm reference count (when CONFIG_ZRAM_MULTI_COMP introduces priority sharing).
- **PAX_MEMORY_SANITIZE** — zero-on-free for `zram_table_entry::handle` (which points to zsmalloc data containing user page bytes), `zcomp_strm::local_copy` and `zcomp_strm::buffer` (which hold per-CPU plaintext during compress/decompress), and the per-zram `zram` slab itself on `zram_remove`.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs `_store` entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `zram_devops` (`.open`, `.release`, `.submit_bio`, `.report_zones`), every `zcomp_ops` (`.create_ctx`, `.destroy_ctx`, `.compress`, `.decompress`, `.setup_params`, `.release_params`), and every sysfs `_show`/`_store` callback marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-zram `zram` / zsmalloc-handle pointer disclosure behind CAP_SYSLOG; suppress `%p` for handles in `block_state` debugfs output (CONFIG_ZRAM_MEMORY_TRACKING).
- **GRKERNSEC_DMESG** — restrict writeback-failure, compression-failure, and pool-exhausted banners to CAP_SYSLOG so attackers cannot probe compression ratio of swap content via dmesg.
- **CAP_SYS_ADMIN on writeback / backing_dev / reset / disksize / mem_limit** — every mutating sysfs entry under `/sys/block/zramN/` requires CAP_SYS_ADMIN; refuse unprivileged writers.
- **Comp-algo PAX_RAP** — `zcomp_ops` is the most-called indirect-call site in the driver; kCFI-typed dispatch is mandatory to prevent function-pointer corruption from cascading through compress/decompress (which run with attacker-controlled page bytes as input).
- **Page-pool PAX_REFCOUNT** — zsmalloc handle reference counts (covered in `mm/zsmalloc.md`) are saturating; defense against double-free of the per-slot handle on parallel write+reset races.
- **Writeback CAP_SYS_ADMIN** — `writeback_store`, `backing_dev_store`, and `wb_limit_store` require CAP_SYS_ADMIN; the backing file is opened with `O_EXCL` and the opening credentials checked against the zram device's owning user namespace.
- **Key-wipe on reset** — `zram_reset_device` walks every slot and `memset_explicit`s any decompression scratch + the per-CPU `zcomp_strm::buffer` and `local_copy` before tearing down `zcomp`; defense against post-reset slab reuse leaking swap contents.
- **Backing-file path validation** — `backing_dev_store` refuses paths under procfs/sysfs/devtmpfs/cgroupfs and refuses pipe/socket file types via `S_ISREG` check.
- **Per-CPU stream isolation** — `zcomp_stream_get` returns a per-CPU stream with preemption disabled via `local_lock`; defense against cross-CPU stream interleaving in compress/decompress.

Rationale: zram stores compressed swap and tmpfs content in kernel memory — every per-slot handle, every per-CPU compression buffer, and every backing-device write contains user data that often includes process memory (since the primary use case is swap). A relaxed slab-sanitize, a writable `zcomp_ops`, a missed CAP_SYS_ADMIN on writeback, or a leaked `zcomp_strm::buffer` exposes the contents of swap to a privileged-but-unauthorised attacker (or worse, to a sibling user namespace). RAP/kCFI on `zcomp_ops`, CAP_SYS_ADMIN on every mutating sysfs, MEMORY_SANITIZE on key-bearing slabs and per-CPU buffers, explicit memset on reset, and refcount-overflow trap on backing-file references turn zram from "compressed swap" into a structural confidentiality boundary.

## Open Questions

- [ ] Q1: Should backing-device writeback default to `compressed_writeback=1` so the disk-resident form remains compressed (preserving size advantage) but at the cost of needing the same algorithm on read-back?
- [ ] Q2: Is the same-page-filling optimisation a side channel — a Spectre-style attacker can probe `mm_stat::same_pages` to learn how many of victim's swap pages are zeroed?
- [ ] Q3: Should `recomp_algorithm` recompression be restricted to idle slots only (CONFIG_ZRAM_TRACK_ENTRY_ACTIME required), or also permitted on active slots with explicit per-write-down opt-in?

## Verification

- `modprobe zram num_devices=2 && echo 1G > /sys/block/zram0/disksize && mkswap /dev/zram0 && swapon /dev/zram0`.
- `cat /sys/block/zram0/{mm_stat,bd_stat,io_stat,comp_algorithm}` reports plausible values.
- `dd if=/dev/urandom of=/tmp/random bs=1M count=64` then `cat /tmp/random > /dev/zram0`; verify `compr_data_size` ~ `orig_data_size` (random does not compress).
- `dd if=/dev/zero of=/dev/zram0 bs=1M count=64` then `same_pages` increments by 64*256 (assuming 4 KB pages) and `compr_data_size` stays near zero.
- `echo 1 > /sys/block/zram0/reset` (with zero openers) returns -EBUSY if mounted, success if unused.
- `blktests` block/zram subset passes.
- `dmesg | grep -i zram` after writeback to a configured backing dev.
- ftrace `block:block_bio_*` events while running fio against `/dev/zram0`.
