# Subsystem: block/ — block I/O layer

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: in-v0
upstream-paths:
  - block/
  - block/partitions/
  - include/linux/blkdev.h
  - include/linux/blk_types.h
  - include/linux/blk-mq.h
  - include/linux/bio.h
  - block/elevator.c
  - include/linux/blk-cgroup.h
  - include/linux/blk-crypto.h
  - include/linux/blk-integrity.h
  - include/uapi/linux/fs.h
  - include/uapi/linux/blkpg.h
  - include/uapi/linux/blktrace_api.h
  - include/uapi/linux/sed-opal.h
-->

## Summary
Tier-2 overview for `block/` — the block I/O layer. Owns the multi-queue request layer (blk-mq), the bio (block I/O request) data path, the request-queue + gendisk + bdev + partition model, three I/O schedulers (mq-deadline, kyber, bfq), block-cgroup throttling and accounting (throttle + iolatency + iocost + ioprio), block-layer crypto (inline encryption + fallback), bio integrity (T10-PI, fs-verity backing), zoned-device support (`blk-zoned.c`), Self-Encrypting-Drive Opal management, and the partition-table parser dispatch (msdos, gpt-efi, mac, amiga, ldm, …).

`block/` is medium-sized (57 top-level files + the `partitions/` subdir) but compat-critical: every disk filesystem and many drivers (NVMe, virtio-blk, scsi, mmc, loop, dm, md) plug into the block layer.

## Upstream references in scope

| Category | Upstream paths | Planned Tier-3 doc |
|---|---|---|
| blk-mq core (multi-queue request layer) | `block/blk-mq.c`, `block/blk-mq.h`, `block/blk-mq-tag.c`, `block/blk-mq-sched.c`, `block/blk-mq-sysfs.c`, `block/blk-mq-debugfs.c`, `block/blk-mq-cpumap.c`, `block/blk-mq-dma.c` | `blk-mq.md` |
| Request queue + gendisk + bdev | `block/blk-core.c`, `block/blk-flush.c`, `block/blk-settings.c`, `block/blk-sysfs.c`, `block/blk-rq-qos.c`, `block/blk-stat.c`, `block/blk-ia-ranges.c`, `block/blk-pm.c`, `block/genhd.c`, `block/bdev.c`, `block/fops.c`, `block/bsg.c`, `block/bsg-lib.c`, `include/linux/blkdev.h`, `include/linux/blk_types.h`, `include/linux/blk-mq.h` | `request-queue.md` |
| bio (block I/O request) | `block/bio.c`, `block/blk-merge.c`, `block/blk-map.c`, `block/blk-lib.c`, `include/linux/bio.h` | `bio.md` |
| ioctl + ABI | `block/ioctl.c`, `include/uapi/linux/{fs,blkpg,blktrace_api}.h` | `block-ioctl.md` |
| Partitions | `block/partitions/` (core dispatcher + msdos / efi GPT / mac / amiga / atari / ibm / ldm / sgi / sun / acorn / aix / karma / osf / sysv68 / ultrix / cmdline / of) | `partitions.md` |
| I/O schedulers | `block/elevator.c`, `block/mq-deadline.c`, `block/kyber-iosched.c`, `block/bfq-iosched.c`, `block/bfq-cgroup.c`, `block/bfq-wf2q.c` | `io-schedulers.md` |
| Block cgroup throttling + iolatency + iocost + ioprio + ioc | `block/blk-cgroup.c`, `block/blk-cgroup-rwstat.c`, `block/blk-cgroup-fc-appid.c`, `block/blk-throttle.c`, `block/blk-iolatency.c`, `block/blk-iocost.c`, `block/blk-ioprio.c`, `block/blk-ioc.c`, `include/linux/blk-cgroup.h` | `cgroup-throttle.md` |
| Block-layer encryption | `block/blk-crypto.c`, `block/blk-crypto-fallback.c`, `block/blk-crypto-profile.c`, `block/blk-crypto-sysfs.c`, `include/linux/blk-crypto.h` | `blk-crypto.md` |
| Bio integrity (T10-PI Data Integrity Field) | `block/blk-integrity.c`, `block/bio-integrity.c`, `block/bio-integrity-auto.c`, `block/bio-integrity-fs.c`, `block/t10-pi.c`, `include/linux/blk-integrity.h` | `bio-integrity.md` |
| Zoned device support | `block/blk-zoned.c` | `zoned.md` |
| Self-Encrypting-Drive (Opal) management | `block/sed-opal.c`, `block/opal_proto.h`, `include/uapi/linux/sed-opal.h` | `sed-opal.md` |
| Bad-block tracking | `block/badblocks.c` | folded into `request-queue.md` |
| Misc Tests | `block/test_*.c` (if present) | (not separately documented) |

## Compatibility contract

block/ owns the following userspace-visible compat surface:

### Syscall surface (block consumes; doesn't own outright)

Block-relevant syscalls are mostly fs/-owned (`open`, `read`, `write`, `pread64`, `pwrite64`, `fsync`, `fdatasync`, `sync_file_range`, `ioctl`); block/ owns the *ioctl numbers and semantics* exposed through block-device file descriptors:

- **`BLKROSET`, `BLKROGET`** — set/get read-only
- **`BLKGETSIZE`, `BLKGETSIZE64`** — device size
- **`BLKSSZGET`, `BLKBSZGET`, `BLKBSZSET`** — sector / block sizes
- **`BLKDISCARD`, `BLKZEROOUT`, `BLKSECDISCARD`** — TRIM / zero / secure-discard
- **`BLKPG`** — partition table management
- **`BLKRRPART`** — re-read partition table
- **`BLKFLSBUF`** — flush buffers
- **`BLKFRAGET`, `BLKFRASET`** — read-ahead size
- **`BLKZNAME`, `BLKGETZONEFRAG`, `BLKZNRZONES`, `BLKZNRESETZONE`, `BLKZNCLOSEZONE`, `BLKZNFINISHZONE`, `BLKZNOPENZONE`, `BLKZNRESETZONE`, `BLKREPORTZONE`** — zoned-device control
- **`BLKTRACESETUP`, `BLKTRACESTART`, `BLKTRACESTOP`, `BLKTRACETEARDOWN`** — blktrace setup (cross-ref `kernel/trace/blktrace.md`)
- **`SED_*` ioctls** — Opal Self-Encrypting-Drive commands
- **`BLKOPENZONE`, `BLKFINISHZONE`, `BLKRESETZONE`** — zoned-device control aliases

Each gets a Tier-5 `uapi/syscalls/` (technically uapi/ioctls/) doc in Phase D.

### `/proc` surfaces

| Path | Owner doc | Compat level |
|---|---|---|
| `/proc/diskstats` | `request-queue.md` | Format-identical |
| `/proc/partitions` | `partitions.md` | Format-identical |
| `/proc/<pid>/io` | `request-queue.md` (cross-ref) | Field-identical |

### `/sys` surfaces

| Path | Owner doc | Compat level |
|---|---|---|
| `/sys/block/<dev>/{queue/*,size,ro,removable,...}` | `request-queue.md` | File set + value semantics identical |
| `/sys/block/<dev>/queue/{scheduler,nr_requests,read_ahead_kb,max_sectors_kb,rotational,nomerges,...}` | `request-queue.md`, `io-schedulers.md` | Identical |
| `/sys/block/<dev>/queue/zoned`, `/sys/block/<dev>/queue/nr_zones`, `/sys/block/<dev>/queue/zone_*` | `zoned.md` | Identical |
| `/sys/block/<dev>/<partition>/{start,size,...}` | `partitions.md` | Identical |
| `/sys/fs/cgroup/<grp>/{io.{stat,max,latency,weight,prio.class},...}` | `cgroup-throttle.md` | Identical (cgroup v2 io controller) |
| `/sys/block/<dev>/queue/crypto/*` | `blk-crypto.md` | Identical |
| `/sys/block/<dev>/integrity/*` | `bio-integrity.md` | Identical |

### Wire-level / on-disk

- **Partition tables**: byte-identical parse semantics for MBR/MSDOS, EFI GPT, Mac, Amiga RDB, Atari, IBM (s390 DASD), LDM (Windows dynamic disk), and the others. Existing partitioning tools (`fdisk`, `gdisk`, `parted`) operate on existing disks unchanged.
- **T10 Protection Information** (DIF / DIX): byte-identical generation and verification — block layer must produce/consume the same 8-byte PI field as upstream so end-to-end checksumming with capable hardware works.
- **Inline Encryption Engine (IEE) command formats**: byte-identical wrapped-key descriptors across the bio-IEE-driver path; an inline-encrypted block written by upstream must be read correctly by Rookery (and vice versa) given the same wrapping key.
- **Opal command frames**: byte-identical SP / Authenticate / Activate / SetSP_Lifecycle / Genkey / RevertSP / Random / GetACL flows.

## Requirements

- REQ-1: blk-mq is the single block submission path — Rookery does not preserve the legacy single-queue request_fn API (already removed upstream by 5.0).
- REQ-2: Every block ioctl listed above is implemented byte-identically with upstream — number, struct layout, semantics.
- REQ-3: bio data structure layout is preserved at the level required for drivers (drivers in `drivers/block/` and `drivers/scsi/`, etc., access `struct bio` fields by macro; field offsets for the macro-accessed cache-line-zero fields must match upstream).
- REQ-4: Three I/O schedulers (mq-deadline, kyber, bfq) preserve their userspace-visible behavior (per-device default, per-device override via `/sys/block/<dev>/queue/scheduler`, tunable knobs).
- REQ-5: Partition table parsers preserve the byte-identical content of `/sys/block/<dev>/<partition>/{start,size,...}` for any disk image that mounts under upstream.
- REQ-6: blkcgroup v1 and v2 controllers (io, blkio) preserve their userspace-visible knobs and accounting semantics.
- REQ-7: block-layer crypto: fscrypt-via-IEE chains identically; the bio's key descriptor is preserved across submission.
- REQ-8: bio-integrity with T10-PI generates and verifies per-block 8-byte CRC/Type/Reference-Tag fields identically to upstream.
- REQ-9: Zoned-device commands (`REQ_OP_ZONE_*`) and the userspace ioctls map identically; existing `zonefs` and SMR-disk userspace tools work unmodified.
- REQ-10: SED Opal management ABI preserved.
- REQ-11: blktrace ABI preserved (relayfs file format, BLKTRACE* ioctls, `/sys/kernel/debug/block/<dev>/trace0` records).
- REQ-12: All Tier-3 docs spawned by this overview each declare their unsafe-block clusters, TLA+ models (where novel concurrency primitives are introduced), and Kani harnesses for data-structure invariants.
- REQ-13: The implementation reuses upstream rust-for-linux's existing block abstractions (`rust/kernel/block.rs`, `rust/kernel/block/`).

## Acceptance Criteria

- [ ] AC-1: A grep over Rookery for "single-queue request_fn" or `blk_init_queue` returns zero matches. (covers REQ-1)
- [ ] AC-2: An ioctl golden-test exercises every BLK* ioctl on Rookery and upstream with curated inputs and reports byte-identical responses. (covers REQ-2)
- [ ] AC-3: A driver simulating a block device (e.g., null_blk, brd) loads identically to upstream and `bio` field offsets match per `pahole` output. (covers REQ-3)
- [ ] AC-4: For each of mq-deadline, kyber, bfq: a fio workload exhibits the same throughput and latency distribution (within ~5%) as upstream. (covers REQ-4)
- [ ] AC-5: Mounting a curated set of disk images (msdos, GPT, Mac, Amiga, IBM/DASD) on Rookery reports identical partitions to upstream. (covers REQ-5)
- [ ] AC-6: cgroup v2 io selftests under `tools/testing/selftests/cgroup/` (the io subset) pass with the same pass/fail set as upstream. (covers REQ-6)
- [ ] AC-7: An fscrypt-encrypted volume backed by an IEE-capable device decrypts identically on Rookery and upstream. (covers REQ-7)
- [ ] AC-8: bio-integrity selftests (or a manual T10-PI simulation harness) verify identical CRC/RefTag generation. (covers REQ-8)
- [ ] AC-9: blkzone selftest output for a zoned-block device under Rookery matches upstream. (covers REQ-9)
- [ ] AC-10: SED Opal lock/unlock cycle on a hardware Opal SSD succeeds on Rookery. (covers REQ-10)
- [ ] AC-11: `blktrace -d /dev/<dev>` produces identical record output (modulo timestamps) under Rookery vs. upstream. (covers REQ-11)
- [ ] AC-12: `make verify` passes block/ Kani harnesses; `make tla` passes block/ models. (covers REQ-12)
- [ ] AC-13: A grep over Rookery for `pub use kernel::block::*` shows reuse of upstream rust-for-linux abstractions. (covers REQ-13)

## Architecture

### Layout map (Tier-3 docs spawned from this overview)

```
.design/block/
  00-overview.md              ← this document
  blk-mq.md                   ← multi-queue request layer (core), tag allocation, dispatch
  request-queue.md            ← request_queue + gendisk + bdev + flush + sysfs + stats + bad-blocks
  bio.md                      ← struct bio + bio chains + map/merge/clone
  block-ioctl.md              ← all BLK* ioctls
  partitions.md               ← partition-table parser dispatch + per-format details (MBR, GPT, Mac, Amiga, …)
  io-schedulers.md            ← elevator + mq-deadline + kyber + bfq
  cgroup-throttle.md          ← blk-cgroup + throttle + iolatency + iocost + ioprio + ioc
  blk-crypto.md               ← inline encryption + fallback + per-device profile
  bio-integrity.md            ← T10-PI generation + verification + bio-integrity-auto + bio-integrity-fs
  zoned.md                    ← zoned-device support (REQ_OP_ZONE_*, zone management)
  sed-opal.md                 ← Self-Encrypting-Drive Opal management
```

### Cross-references

- `mm/00-overview.md` — page-cache integration: bio submission consumes folios; writeback consumes mm/'s page-writeback.
- `fs/00-overview.md` — `iomap.md` is the modern fs↔block bridge; per-FS docs (ext4, btrfs, xfs) consume blk-mq.
- `kernel/00-overview.md` — `kernel/cgroup/` framework substrate + `kernel/trace/blktrace.md` cross-reference.
- `drivers/00-overview.md` (Phase B) — every block driver (drivers/block/, drivers/nvme/, drivers/scsi/, drivers/md/, drivers/ata/, drivers/mmc/) consumes blk-mq's API.
- `crypto/00-overview.md` (Phase B) — blk-crypto fallback uses crypto API.
- `00-glossary.md` — `bio`, `blk_mq`, `request_queue`, `gendisk`.

### Rust module organization (informative)

- `kernel::block` — exists upstream (`rust/kernel/block.rs` + `rust/kernel/block/`); extend
- `kernel::block::mq` — already partly exists in upstream
- `kernel::block::bio` — Rookery to author / extend
- `kernel::block::cgroup` — Rookery to author
- `kernel::block::crypto` — Rookery to author
- `kernel::block::integrity` — Rookery to author
- `kernel::block::zoned` — Rookery to author
- `kernel::block::partitions` — Rookery to author
- `kernel::block::scheduler::{deadline, kyber, bfq}` — Rookery to author

### Locking and concurrency

- `request_queue` lock (`q->queue_lock`) — protects request lifecycle.
- Per-`hctx` (hardware queue) tag bitmap — atomic ops + waitqueue per-tag.
- Per-`request` lifecycle is RCU-managed (BIO_REM and BIO_DONE transitions use atomics).
- bio_set's bio mempool — backed by `mempool` (mm-owned).
- `blkcg` hierarchy — RCU-protected child list, parent counter rollups.

### Error handling

block/ returns `Result<T, KernelError>` where applicable. Common returns:
- `Err(EIO)` — generic I/O error
- `Err(ETIMEDOUT)` — request timed out
- `Err(EOPNOTSUPP)` — operation not supported by device (TRIM on non-supporting drive)
- `Err(EREMOTEIO)` — driver-reported error
- `Err(ENOMEM)` — bio allocation failed
- `Err(ENOLINK)` — device disconnected mid-request
- `Err(EILSEQ)` — bad data integrity

## Verification

### Layer 1: Kani SAFETY proofs

Anticipated `unsafe` clusters:
- bio scatter-list construction in `kernel::block::bio::*`
- DMA-mapped bio transfers in `kernel::block::mq::dma::*`
- request lifecycle transitions in `kernel::block::mq::*`
- partition entry parsing for each format in `kernel::block::partitions::*`
- Opal command-frame construction in `kernel::block::sed_opal::*`

### Layer 2: TLA+ models (mandatory for novel concurrency)

- `models/block/blk_mq_tag_alloc.tla` — proves tag allocation under concurrent submission across hctxs never double-allocates.
- `models/block/request_state_machine.tla` — proves request state transitions (IDLE → SUBMITTED → IN_FLIGHT → DONE / TIMEOUT / FAILED) honor invariants under race between dispatch, timeout, and completion.
- `models/block/blkcg_accounting.tla` — proves blk-cgroup hierarchical accounting consistency under concurrent bio submission across descendant cgroups.
- `models/block/bio_integrity_chain.tla` — proves the bio-integrity chain (data + integrity in lock-step) preserves ordering across split/merge.
- `models/block/elevator.tla` — proves I/O scheduler ordering invariants for each of mq-deadline / kyber / bfq.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Tag bitmap (per hctx) | "Each tag is in exactly one of: free, allocated, reserved" | `kani::proofs::block::tag_invariants` |
| Request linked list | "No request appears twice in any list (free, dispatch, timeout, complete)" | `kani::proofs::block::request_list_invariants` |
| bio chain | "bi_next chain is acyclic; the chain length matches bi_iter consumption" | `kani::proofs::block::bio_chain_invariants` |
| blkcg hierarchy | "Sum of children's counters ≤ parent's counter" | `kani::proofs::block::blkcg_accounting_invariants` |

### Layer 4: Functional correctness via Creusot / Verus / Prusti (opt-in)

Strong opt-in candidates:
- `partitions.md` — partition-table parsers are file-format parsers (small, well-defined). Creusot proofs of "valid input → valid parse" are tractable.
- `bio-integrity.md` — T10-PI CRC/Type/RefTag generation is deterministic spec-defined. Verus/Creusot proofs are tractable.
- `blk-crypto.md` — IEE key-descriptor derivation deterministic; Verus/Creusot tractable.

## Hardening

Placeholder per `00-overview.md` D6. block/ owns implementation of:

- Block-layer crypto (above) — protects data at rest with hardware-IEE fallback to software.
- Bio integrity (T10-PI) — detects silent data corruption.
- SED Opal — drive-level encryption.
- BLKSECDISCARD — secure-erase semantics.

Per D6, binding policy in deferred `00-security-principles.md`.

## Open Questions

(none — all decisions baked from baseline; Q's surface in Tier-3 docs as components are detailed)

## Out of Scope

- Legacy single-queue `request_fn` API (already removed upstream).
- 32-bit-only paths (consistent with arch v0).
- Test fixtures.
- Implementation code — `.design/` contains specs only.
