---
title: "Tier-5 syscall: cachestat(2) — syscall 451"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`cachestat(2)` reports per-page-cache statistics over a byte range of a file descriptor: how many pages are present in the page cache, how many are dirty, how many are pending writeback, how many are in the active LRU list (recently referenced), and how many were evicted-then-recently-rebrought-in. It produces a `struct cachestat` snapshot that user-space tools (cachestat from BCC, `vmtouch`, performance tooling) use to reason about working-set fit, dirty-page pressure, and re-faulting.

Unlike `mincore(2)` (per-page presence bitmap), `cachestat(2)` returns aggregate counts, so it scales to gigabyte-range queries without the userspace memory or kernel time cost of a per-page byte array.

Critical for: database working-set introspection, file-server cache-pressure dashboards, `posix_fadvise`-decision feedback loops, the BCC `cachestat` tool replacing the per-process eBPF-based estimator.

### Acceptance Criteria

- [ ] AC-1: Read 4KB from a 1MB file then `cachestat(off=0, len=1MB)`: `nr_cache >= 1`.
- [ ] AC-2: `fsync(fd)` then `cachestat`: `nr_dirty == 0`, `nr_writeback == 0`.
- [ ] AC-3: `posix_fadvise(POSIX_FADV_DONTNEED)` then `cachestat`: counts go to zero.
- [ ] AC-4: Pipe fd: `-EOPNOTSUPP`.
- [ ] AC-5: `flags = 1`: `-EINVAL`.
- [ ] AC-6: `off = UINT64_MAX, len = 1`: `-EINVAL` (overflow).
- [ ] AC-7: Range past i_size: all zeros.
- [ ] AC-8: `len = 0`: covers entire file from `off`.
- [ ] AC-9: Refault after eviction increments `nr_evicted` on next call.
- [ ] AC-10: Shmem with swapped-out pages: `nr_evicted` includes shadow entries.
- [ ] AC-11: Hugepage range counts include hugepage base-page count.

### Architecture

```rust
#[syscall(nr = 451, abi = "sysv")]
pub fn sys_cachestat(
    fd: u32,
    range_p: UserPtr<CachestatRange>,
    cstat_p: UserPtr<Cachestat>,
    flags: u32,
) -> isize {
    Cachestat::query(fd, range_p, cstat_p, flags)
}
```

`Cachestat::query(fd, range_p, cstat_p, flags) -> isize`:
1. if flags != 0 { return Err(EINVAL); }
2. let range: CachestatRange = range_p.read()?;                 // EFAULT
3. let end = range.off.checked_add(range.len).ok_or(EINVAL)?;
4. let file = Fd::lookup(current, fd)?;                         // EBADF
5. let mapping = file.address_space().ok_or(EOPNOTSUPP)?;
6. let mut out = Cachestat::default();
7. let len = if range.len == 0 {
8.     file.size().saturating_sub(range.off)
9. } else {
10.    range.len
11. };
12. Cachestat::walk_mapping(mapping, range.off, len, &mut out)?;
13. cstat_p.write(out)?;                                        // EFAULT zeroes nothing
14. Ok(0)

`Cachestat::walk_mapping(mapping, off, len, out) -> Result<()>`:
1. let start_idx = off / PAGE_SIZE;
2. let end_idx = (off + len + PAGE_SIZE - 1) / PAGE_SIZE;
3. xa_for_each(&mapping.i_pages, start_idx..end_idx, |entry| {
4.    if entry.is_value() {
5.        /* Shadow / workingset entry */
6.        out.nr_evicted += 1;
7.        if workingset_recent(entry, jiffies()) { out.nr_recently_evicted += 1; }
8.    } else if let Some(folio) = entry.as_folio() {
9.        let n = folio.nr_pages();
10.       out.nr_cache += n;
11.       if folio.is_dirty()     { out.nr_dirty     += n; }
12.       if folio.is_writeback() { out.nr_writeback += n; }
13.    }
14. });
15. Ok(())

### Out of Scope

- Page-cache internal data structures (Tier-3 `mm/filemap.md`).
- Workingset refault algorithm (Tier-3 `mm/workingset.md`).
- `mincore(2)` (separate Tier-5 doc).
- Implementation code.

### signature

```c
int cachestat(
    unsigned int fd,
    struct cachestat_range *cstat_range,
    struct cachestat *cstat,
    unsigned int flags
);
```

```c
struct cachestat_range {
    __u64 off;     /* byte offset, may be > i_size */
    __u64 len;     /* bytes; 0 means "to end-of-file" */
};

struct cachestat {
    __u64 nr_cache;            /* pages in page cache */
    __u64 nr_dirty;            /* dirty pages */
    __u64 nr_writeback;        /* pages under writeback */
    __u64 nr_evicted;          /* pages evicted then refaulted (workingset refault) */
    __u64 nr_recently_evicted; /* refaulted within window (active workingset) */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `unsigned int` | in | File descriptor of a regular file, shmem object, or block device that has a page cache. |
| `cstat_range` | `struct cachestat_range *` | in | Byte range over which to aggregate. |
| `cstat` | `struct cachestat *` | out | Receives the five counts. |
| `flags` | `unsigned int` | in | Reserved; MUST be `0`. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; `*cstat` populated. |
| `-1` + `errno` | Failure; `*cstat` unchanged. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `cstat_range` or `cstat` user pointer invalid. |
| `EINVAL` | `flags != 0`, `cstat_range->off + cstat_range->len` overflows `__u64`. |
| `EBADF` | `fd` not a valid descriptor. |
| `EOPNOTSUPP` | `fd` refers to a file type without a page cache (pipe, socket, char device, FIFO, anon-fd). |
| `ENOMEM` | Kernel could not allocate workingset shadow buffer for refault accounting. |

### abi surface

```text
__NR_cachestat  (x86_64)  = 451
__NR_cachestat  (arm64)   = 451
__NR_cachestat  (riscv)   = 451
__NR_cachestat  (i386)    = 451

/* struct cachestat is 5 * 8 = 40 bytes, stable layout. */
/* struct cachestat_range is 2 * 8 = 16 bytes, stable layout. */
```

### compatibility contract

REQ-1: Syscall number is **451** on x86_64. ABI-stable.

REQ-2: `flags` MUST be `0`. No flags defined.

REQ-3: `cstat_range->len == 0` means "to end of file". `off + len` MUST NOT overflow.

REQ-4: `cstat_range->off > i_size` is legal; all counts are zero in that case.

REQ-5: `fd` MUST be a file that supports a page cache: regular file (any filesystem with address-space ops), shmem/tmpfs (anonymous in-memory file), and block-device fd (raw partitions). Pipes, sockets, char devices, eventfd, signalfd, timerfd, perf-fd ⟹ `-EOPNOTSUPP`.

REQ-6: Counts are computed via per-folio walk over the address-space `i_pages` xarray, restricted to the requested range. Hugepages count as their constituent base-page count.

REQ-7: `nr_cache` is the total page count present in cache (including dirty/writeback). It is NOT a sum of the other fields.

REQ-8: `nr_dirty` is the number of pages with `PG_dirty` set; `nr_writeback` is `PG_writeback`. A page can be both, counted in both.

REQ-9: `nr_evicted` is the number of refault entries (shadow nodes) found in the range — pages that were once in cache, evicted, and the eviction shadow still exists. This is the **workingset signal**.

REQ-10: `nr_recently_evicted` is the subset of `nr_evicted` whose shadow age indicates the eviction was recent enough that the page is considered part of the active workingset (would have been retained with more memory).

REQ-11: No special capability is required; the caller must merely hold `fd`.

REQ-12: `cstat` is fully populated on success; on error the user buffer is NOT written.

REQ-13: Concurrent writes to the cache (page-cache add/remove via I/O, writeback, eviction) make this a snapshot, NOT a transactional view. Consecutive calls may differ.

REQ-14: For block-device fds (raw block device), the range is in bytes of the block-device address space; the kernel reads `bdev->bd_inode->i_mapping`.

REQ-15: For shmem fds, `nr_evicted` and `nr_recently_evicted` reflect swap-out shadow entries; the rest are in-memory pages.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_strict_zero` | INVARIANT | `flags != 0` ⟹ `-EINVAL`. |
| `range_no_overflow` | INVARIANT | `off + len` overflow ⟹ `-EINVAL`. |
| `no_cache_no_op` | INVARIANT | pipe / socket fd ⟹ `-EOPNOTSUPP`, no walk attempted. |
| `out_atomic` | INVARIANT | success ⟹ all 5 fields written; failure ⟹ none. |
| `len_zero_eq_to_end` | INVARIANT | `len == 0` ⟹ walk covers `[off, i_size)`. |
| `hugepage_accounting` | INVARIANT | hugepage folio contributes `nr_pages` to counts. |

### Layer 2: TLA+

`mm/cachestat.tla`:
- Per-call transitions: parse, fd-lookup, mapping-walk, write-out.
- Properties:
  - `safety_no_partial_out` — output struct atomic.
  - `safety_no_walk_on_bad_fd` — bad fd ⟹ early exit.
  - `safety_shadow_count` — `nr_evicted` strictly counts shadow xa entries.
  - `liveness_bounded_walk` — walk terminates in `O(end_idx - start_idx)`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Cachestat::query` post: success ⟹ cstat fully populated | `Cachestat::query` |
| `Cachestat::walk_mapping` post: nr_cache == sum of folio nr_pages | `Cachestat::walk_mapping` |
| `Cachestat::walk_mapping` post: shadows counted in nr_evicted only | `Cachestat::walk_mapping` |
| `Cachestat::query` post: nr_dirty <= nr_cache, nr_writeback <= nr_cache | `Cachestat::query` |

### Layer 4: Verus / Creusot functional

Per-`cachestat(2)` man page; BCC `cachestat` tool semantics; vfs/page-cache documentation in `Documentation/admin-guide/mm/`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`cachestat(2)` reinforcement:

- **Per-`flags == 0` strict** — defense against per-future-flag smuggle.
- **Per-`off + len` overflow check** — defense against per-arithmetic-UB.
- **Per-page-cache-only fd allow-list** — defense against per-info-leak on socket/pipe fds.
- **Per-counts sub-aggregate consistency** — defense against per-state-snapshot incoherence.

### grsecurity / pax surface

- **PaX UDEREF on `cstat_range` and `cstat`** — defense against per-pointer smuggling; SMAP forced for both copy_from and copy_to.
- **cachestat info-leak prevention (PAX_USERCOPY)** — `cachestat` output struct is fixed-size 40 bytes; PAX_USERCOPY validates the write hits a whitelisted destination region. No internal kernel-pointer or shadow-cookie ever leaks via the counts (only aggregate page counts).
- **GRKERNSEC_PROC_GETPID file-access policy** — `fd` MUST already have been opened by the caller; the syscall does not bypass the file's open-time DAC/MAC checks (no `cachestat` on `/proc/<other>/mem` etc. unless `fd` was already legitimately obtained).
- **Per-shmem swap-shadow exposure restricted** — `GRKERNSEC_SHMEM_SWAP_HIDE` zeroes `nr_evicted`/`nr_recently_evicted` for shmem fds when caller is not `CAP_SYS_ADMIN`; defense against side-channel sniffing of other-process swap-out behavior on shared tmpfs.
- **PAX_REFCOUNT on `file` refcount during the call** — defense against per-fd-close-race UAF.
- **GRKERNSEC_HIDESYM on `address_space` pointer** — pointer never echoed back to userspace.
- **Per-range bounded walk** — defense against per-DoS via huge range on tiny file; range clamped to `i_size` internally.
- **PAX_USERCOPY destination check on `struct cachestat` write** — bounded `copy_to_user` validated against caller mm.
- **Per-block-device fd capability gate** — `GRKERNSEC_BDEV_CACHESTAT_PRIV` requires `CAP_SYS_RAWIO` to call `cachestat` on a block-device fd (defense against fingerprinting other tenants' I/O patterns on a shared block device).
- **Per-`-EOPNOTSUPP` zero-write** — defense against per-stale-stack-data leak: on error, the user buffer is NOT touched, so old stack contents are not aliased to fresh kernel data.
- **Per-`nr_recently_evicted` jiffies-window randomized per-boot** — `GRKERNSEC_WORKINGSET_JITTER` adds boot-random jitter to the "recent" threshold; defense against per-timing-side-channel.

