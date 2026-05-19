# Tier-5 syscall: readahead(2) — syscall 187

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - mm/readahead.c (SYSCALL_DEFINE3(readahead), ksys_readahead, force_page_cache_readahead)
  - mm/filemap.c (page_cache_sync_readahead)
  - include/linux/fs.h
  - arch/x86/entry/syscalls/syscall_64.tbl (187  common  readahead)
-->

## Summary

`readahead(2)` populates the page cache for a file range without copying data to userspace. The kernel issues asynchronous bio reads for the pages covering `[offset, offset+count)`, returning as soon as the requests are queued. Subsequent `read(2)` / `mmap(2)` accesses to those pages hit the cache, avoiding sync I/O.

`readahead(2)` is a less-flexible cousin of `posix_fadvise(POSIX_FADV_WILLNEED)`: same effect, simpler signature, but advisory in identical fashion (kernel may issue fewer reads under memory pressure). Used by media servers, database warm-ups, package managers, and `vmtouch`. Critical for: I/O latency hiding, cold-start mitigation, large-file streaming.

## Signature

```c
ssize_t readahead(int fd, off64_t offset, size_t count);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor referring to a regular file or block device. |
| `offset` | `off64_t` | in | Starting byte offset into the file. |
| `count` | `size_t` | in | Bytes to prefetch. Kernel rounds to page boundaries and may cap. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Readahead queued (no count of pages — return value is 0, not bytes). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` not a valid open fd; or fd is O_PATH; or fd is write-only (no read mode). |
| `EINVAL` | fd refers to a pipe/socket/FIFO/directory; `offset` negative; arithmetic overflow on `offset + count`; fs/bdev does not support readahead. |
| `ESPIPE` | fd refers to a pipe or FIFO (some kernels return EINVAL). |
| `EIO` | I/O error during readahead is NOT propagated to the syscall (queued async); only pre-queue errors return EIO. |

## ABI surface

```text
__NR_readahead  (x86_64)  = 187
__NR_readahead  (arm64)   = 213
__NR_readahead  (riscv)   = 213
__NR_readahead  (i386)    = 225

/* count may be silently capped to bdi.ra_pages * PAGE_SIZE * some-factor;
   no error on truncation. */
/* readahead returns 0 on success, not the bytes-queued count. */
```

## Compatibility contract

REQ-1: Syscall number is **187** on x86_64. ABI-stable.

REQ-2: `fd` resolved via `fdget(fd)`. Must be FMODE_READ; O_WRONLY → `-EBADF`. O_PATH → `-EBADF`.

REQ-3: Fd type validation:
- Regular file: accepted.
- Block device: accepted.
- Pipe / socket / FIFO / directory / anon-inode: `-EINVAL` or `-ESPIPE`.

REQ-4: `offset < 0` → `-EINVAL`.

REQ-5: `offset + count` arithmetic overflow → `-EINVAL`.

REQ-6: `count == 0` → returns 0 (no work).

REQ-7: Backing-device support:
- `mapping->a_ops->readahead` or `mapping->a_ops->readpages` MUST be present; else `-EINVAL`.

REQ-8: Page-range computation:
- start_page = `offset >> PAGE_SHIFT`.
- end_page   = `(offset + count + PAGE_SIZE - 1) >> PAGE_SHIFT`.
- Capped at `i_size >> PAGE_SHIFT + 1` (cannot prefetch beyond EOF).

REQ-9: Issue readahead via `force_page_cache_readahead(mapping, file, start, nr_pages)`:
- Spawns async bio reads for missing pages.
- Already-cached pages are skipped.
- Memory pressure may reduce the actual range.

REQ-10: Syscall is asynchronous: returns 0 once bios are queued. Errors during bio completion do not propagate.

REQ-11: No capability requirement (any reader of fd may readahead).

REQ-12: Per-`vm.read_ahead_kb` / per-bdi `ra_pages`: caps the effective prefetch window per call (kernel may exceed if explicitly requested, depending on version).

REQ-13: `mlocked`-style prefetch is NOT performed; readahead pages may be reclaimed before use under pressure.

REQ-14: Per-`O_DIRECT` fd: readahead operates on the page cache (used for mixed I/O scenarios). For pure-DIRECT files where cache is bypassed, readahead is a no-op but returns 0.

REQ-15: Equivalent to `posix_fadvise(fd, offset, count, POSIX_FADV_WILLNEED)` minus the f_ra.ra_pages tweaks.

## Acceptance Criteria

- [ ] AC-1: `readahead(fd, 0, 4096)` on regular file: returns 0; subsequent `read` is fast.
- [ ] AC-2: `fd = -1`: `-EBADF`.
- [ ] AC-3: `fd` write-only: `-EBADF`.
- [ ] AC-4: `fd` on pipe: `-EINVAL` (or `-ESPIPE`).
- [ ] AC-5: `fd` on directory: `-EINVAL`.
- [ ] AC-6: `offset = -1`: `-EINVAL`.
- [ ] AC-7: `offset = i64::MAX, count = 1`: `-EINVAL`.
- [ ] AC-8: `count = 0`: returns 0 (no work).
- [ ] AC-9: `readahead` on a tmpfs file: returns 0 (no-op-ish).
- [ ] AC-10: `readahead` on O_DIRECT-only fs (e.g., raw device with no cache): returns 0; data not cached.
- [ ] AC-11: Readahead of 1 GiB on low-memory system: returns 0; kernel issues bounded fraction.
- [ ] AC-12: `readahead` on filesystem without readpages/readahead op: `-EINVAL`.

## Architecture

```rust
#[syscall(nr = 187, abi = "sysv")]
pub fn sys_readahead(fd: i32, offset: i64, count: usize) -> isize {
    Readahead::do_readahead(fd, offset, count)
}
```

`Readahead::do_readahead(fd, offset, count) -> isize`:
1. let file = fdget(fd).ok_or(EBADF)?;
2. if file.is_o_path() { return -EBADF; }
3. if !file.mode.contains(FMODE_READ) { return -EBADF; }
4. if offset < 0 { return -EINVAL; }
5. if count == 0 { return 0; }
6. let end = match (offset as u64).checked_add(count as u64) {
7.   Some(e) => e,
8.   None    => return -EINVAL,
9. };
10. let inode = file.inode();
11. if inode.is_pipe() || inode.is_fifo() { return -ESPIPE; }
12. if inode.is_socket() || inode.is_dir() { return -EINVAL; }
13. if !inode.is_regular() && !inode.is_blkdev() { return -EINVAL; }
14. let mapping = file.mapping();
15. if mapping.a_ops.readahead.is_none() && mapping.a_ops.readpages.is_none() { return -EINVAL; }
16. let isize = i_size_read(mapping.host);
17. let cap = (isize as u64 + PAGE_SIZE as u64 - 1) >> PAGE_SHIFT;
18. let start_pg = (offset as u64) >> PAGE_SHIFT;
19. let end_pg = min(end >> PAGE_SHIFT, cap);
20. if end_pg <= start_pg { return 0; }
21. let nr_pages = (end_pg - start_pg) as usize;
22. force_page_cache_readahead(mapping, &file, start_pg, nr_pages);
23. 0

`force_page_cache_readahead(mapping, file, start, nr_pages)`:
1. let bdi = mapping.backing_dev_info();
2. let max = min(nr_pages, bdi.ra_pages.max(1) * READAHEAD_FORCE_FACTOR);
3. let ractl = ReadaheadControl::new(file, mapping, start, max);
4. mapping.a_ops.readahead(&ractl)
5.   .or_else(|| do_page_cache_ra(&ractl, max));

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_validated_readable` | INVARIANT | EBADF if !FMODE_READ ∨ O_PATH ∨ invalid. |
| `offset_non_negative` | INVARIANT | offset ≥ 0. |
| `overflow_checked` | INVARIANT | offset + count overflow ⟹ EINVAL. |
| `fd_type_validated` | INVARIANT | pipe/socket/fifo/dir/anon ⟹ EINVAL/ESPIPE. |
| `mapping_supports_ra` | INVARIANT | a_ops.readahead ∨ readpages required. |
| `bounded_by_isize` | INVARIANT | end_page ≤ ceil(i_size / PAGE_SIZE). |

### Layer 2: TLA+

`mm/readahead.tla`:
- States: per-fdget, per-validate-range, per-mapping-check, per-issue-ra, per-return.
- Properties:
  - `safety_no_prefetch_past_eof`.
  - `safety_overflow_rejected`.
  - `safety_async_no_error_propagation`.
  - `safety_count_zero_noop`.
  - `liveness_returns_zero_or_error`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_readahead` post: ret 0 ⟹ readahead queued or no-op | `Readahead::do_readahead` |
| `force_page_cache_readahead` post: ≤ bdi.ra_pages * factor pages queued | `Readahead::force_page_cache_readahead` |
| `i_size_cap` post: nothing queued past i_size | `Readahead::do_readahead` |
| `mapping_ops_present` post: EINVAL if no readahead op | `Readahead::do_readahead` |

### Layer 4: Verus / Creusot functional

Per-`readahead(2)` man-page and per-`mm/readahead.c` semantic equivalence. LTP `readahead01..03` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`readahead(2)` reinforcement:

- **Per-fd FMODE_READ strict** — defense against per-write-only-fd misuse.
- **Per-range overflow check** — defense against per-offset+count arithmetic overflow.
- **Per-mapping-ops required** — defense against per-NULL-op dereference.
- **Per-i_size cap** — defense against per-prefetch-beyond-EOF resource waste.
- **Per-memory-pressure bound** — defense against per-readahead-induced OOM.

## Grsecurity / PaX surface

- **PaX UDEREF none applicable** — readahead takes no user pointers; args are fd/offset/count. No SMAP-relevant copy_from_user.
- **fadvise EBADF strict (shared rule)** — fdget validation matches fadvise: closed fd, O_PATH, write-only, anon-inode-without-readahead-op return `-EBADF` immediately. Defense against per-fd-confusion downstream.
- **CAP_SYS_ADMIN gate on cross-uid readahead of mlocked files** — under GRKERNSEC_READAHEAD_HARDEN, a non-owner caller issuing readahead against an mlocked file region requires `CAP_SYS_ADMIN`. Defense against per-cache-warming side-channel attacks where one tenant primes another's working set.
- **GRKERNSEC_TPE-correlation** — Trusted-Path Execution: readahead on TPE-owned binaries by non-trusted callers is rate-limited; defense against per-readahead-induced text-cache priming (Spectre-v1-style timing attack feasibility).
- **GRKERNSEC_HIDESYM on bdi.ra_pages and ra_pages_max** — readahead window dimensions exposed via /sys gated; defense against per-window-leak used to fingerprint backing device.
- **GRKERNSEC_READAHEAD_RATE_LIMIT** — readahead calls per-uid rate-limited (default 10k/sec); exceeding triggers throttle + audit. Defense against per-bio-flood DoS.
- **PAX_REFCOUNT on file and mapping refcount** — defense against per-refcount-overflow UAF on concurrent readahead + close race.
- **GRKERNSEC_AUDIT_READAHEAD** — readahead calls on cross-uid files, on files marked `noatime+integrity-measured`, or on block devices logged with PID, comm, path, offset, count.
- **kernel_lockdown integrity-mode** — readahead on the kernel-image bdev (`/dev/sda` containing boot partition under integrity policy) is gated; defense against per-integrity-measurement-cache priming.
- **GRKERNSEC_KSTACKOVERFLOW** — defense against per-fs `a_ops->readahead` callchain stack overflow.
- **a_ops->readahead hook validation** — under hardened mode, only built-in fs `readahead` hooks are accepted; out-of-tree modules cannot register custom hooks.
- **PAX_USERCOPY_HARDEN not directly relevant** — no userspace copy on this path.
- **GRKERNSEC_BPF_HARDEN correlation** — readahead tracepoints (`syscalls:sys_enter_readahead`) gated by `CAP_PERFMON` + perf_paranoid level under hardened mode.
- **Bdi bound check** — under hardened mode the bdi-cap factor (`READAHEAD_FORCE_FACTOR`) is reduced from kernel default to a smaller cap to limit per-syscall I/O burst.
- **Per-cgroup I/O accounting** — readahead bytes attributed to caller's blkcg; defense against per-readahead-induced cross-cgroup I/O starvation.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Readahead window algorithm and heuristics (covered in Tier-3 `mm/readahead.md`).
- bio submission and bdev async I/O (covered in Tier-3 `block/blk-mq.md`).
- Per-fs `a_ops->readahead` (covered in per-fs Tier-3 docs).
- Implementation code.
