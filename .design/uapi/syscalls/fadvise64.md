# Tier-5 syscall: fadvise64(2) — syscall 221

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - mm/fadvise.c (SYSCALL_DEFINE4(fadvise64), generic_fadvise)
  - mm/readahead.c (force_page_cache_readahead)
  - include/uapi/linux/fadvise.h (POSIX_FADV_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (221  common  fadvise64)
-->

## Summary

`fadvise64(2)` declares an application's intended access pattern over a file range, allowing the kernel to optimize page-cache prefetch, eviction, and readahead window. Advice values:
- `POSIX_FADV_NORMAL` — reset to default heuristic.
- `POSIX_FADV_RANDOM` — disable readahead.
- `POSIX_FADV_SEQUENTIAL` — double readahead.
- `POSIX_FADV_WILLNEED` — prefetch the range into the page cache.
- `POSIX_FADV_DONTNEED` — drop clean pages in the range from the cache.
- `POSIX_FADV_NOREUSE` — currently a no-op (historically: drop after read).

`fadvise64(2)` is the kernel's voluntary cache-policy hook; database engines (PostgreSQL, MySQL), backup tools (rsync, bup), and media players use it heavily. Critical for: page-cache hygiene, latency-sensitive I/O, low-memory workload patterns.

## Signature

```c
int fadvise64(int fd, off_t offset, off_t len, int advice);
```

```c
#define POSIX_FADV_NORMAL       0
#define POSIX_FADV_RANDOM       1
#define POSIX_FADV_SEQUENTIAL   2
#define POSIX_FADV_WILLNEED     3
#define POSIX_FADV_DONTNEED     4
#define POSIX_FADV_NOREUSE      5
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor. Must reference a regular file (or block device). |
| `offset` | `off_t` | in | Starting byte offset into the file. |
| `len` | `off_t` | in | Length in bytes. `0` means "from offset to end of file". |
| `advice` | `int` | in | One of `POSIX_FADV_*`. Unknown values → `-EINVAL`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Advice accepted (kernel best-effort applies). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` not a valid open fd. |
| `EINVAL` | Unknown `advice`; `offset` or `len` negative; arithmetic overflow `offset + len`; fd refers to a pipe/socket/FIFO; fd is O_PATH. |
| `ESPIPE` | fd refers to a pipe or FIFO (some kernels), often folded into `EINVAL`. |

## ABI surface

```text
__NR_fadvise64    (x86_64)  = 221
__NR_fadvise64_64 (i386)    = 272
__NR_fadvise64    (arm64)   = 223
__NR_fadvise64    (riscv)   = 223

/* On 32-bit ABIs the syscall is fadvise64_64 with 64-bit offset/len
   split into two halves; this design covers the 64-bit form. */
/* len == 0 is interpreted as "until end of file" per POSIX. */
```

## Compatibility contract

REQ-1: Syscall number is **221** on x86_64. ABI-stable.

REQ-2: `fd` resolved via `fdget(fd)`. Returns `-EBADF` for closed/never-opened/O_PATH fds. fdget strict: invalid fd never proceeds to advice dispatch.

REQ-3: Fd type validation:
- Regular file or block device: accepted.
- Pipe / FIFO / socket / anon-inode: `-ESPIPE` or `-EINVAL`.
- Directory: `-EINVAL`.

REQ-4: `offset < 0` or `len < 0` → `-EINVAL`.

REQ-5: `offset + len` overflow (checked via `check_add_overflow`) → `-EINVAL`.

REQ-6: `len == 0` → range is `[offset, i_size_read(inode))`.

REQ-7: `advice` dispatch:
- `NORMAL`:   set `file->f_ra.ra_pages = bdi->ra_pages` (default).
- `RANDOM`:   `file->f_ra.ra_pages = 0` (disable readahead).
- `SEQUENTIAL`: `file->f_ra.ra_pages = 2 * bdi->ra_pages` (double).
- `WILLNEED`: call `force_page_cache_readahead(mapping, file, start, nr_pages)` to populate cache.
- `DONTNEED`: invalidate clean pages in `[offset, offset+len)` via `invalidate_mapping_pages`. Dirty pages may be skipped.
- `NOREUSE`:  no-op (kept for ABI; some patches make it behave like DONTNEED + post-read drop).

REQ-8: `POSIX_FADV_DONTNEED`:
- Round `offset` up to page boundary; `offset + len` down to page boundary.
- Pages in `[start_idx, end_idx)` invalidated if clean.
- Dirty pages may be written back asynchronously (if `bdi->capabilities & BDI_CAP_WRITEBACK`) then invalidated; or skipped entirely (no error).

REQ-9: `POSIX_FADV_WILLNEED`:
- Bounded by `nr_pages` derived from `len` and `bdi->ra_pages` (kernel may cap to avoid memory pressure).
- Async prefetch; syscall returns immediately.
- No error if pages are already present.

REQ-10: `fadvise64(2)` is a hint; the kernel MAY ignore advice (e.g., under memory pressure WILLNEED may prefetch fewer pages, DONTNEED may skip pages locked in mlock).

REQ-11: No capability requirement (any caller with a valid fd may advise).

REQ-12: Per-fs hook: `inode->i_fop->fadvise` may override generic behavior (e.g., tmpfs, hugetlbfs, fuse).

REQ-13: fadvise on shmem/tmpfs:
- WILLNEED is a no-op (data already in cache).
- DONTNEED may free swap entries.

REQ-14: fadvise on `O_DIRECT` fd: advice still applies to the file's page cache (used by mixed-mode I/O); WILLNEED is a no-op when no buffered I/O is expected.

REQ-15: Per-`vm.dirty_background_ratio` / `vm.dirty_ratio` not directly affected; fadvise does not flush dirty data (use `fsync(2)` / `sync_file_range(2)`).

## Acceptance Criteria

- [ ] AC-1: `fadvise64(fd, 0, 0, POSIX_FADV_NORMAL)` on regular file: returns 0.
- [ ] AC-2: `fd = -1`: `-EBADF`.
- [ ] AC-3: `fd` on pipe: `-ESPIPE` (or `-EINVAL`).
- [ ] AC-4: `advice = 99`: `-EINVAL`.
- [ ] AC-5: `offset = -1`: `-EINVAL`.
- [ ] AC-6: `len = -1`: `-EINVAL`.
- [ ] AC-7: `offset = i64::MAX, len = 1`: `-EINVAL` (overflow).
- [ ] AC-8: `WILLNEED` on cold file: pages later observed in cache via `/proc/<pid>/pagemap`.
- [ ] AC-9: `DONTNEED` on cached clean range: pages invalidated.
- [ ] AC-10: `DONTNEED` on cached dirty range: pages may remain (no error).
- [ ] AC-11: `RANDOM` then read: minimal readahead observed.
- [ ] AC-12: `SEQUENTIAL` then read: doubled readahead observed.

## Architecture

```rust
#[syscall(nr = 221, abi = "sysv")]
pub fn sys_fadvise64(fd: i32, offset: i64, len: i64, advice: i32) -> isize {
    Fadvise::do_fadvise64(fd, offset, len, advice)
}
```

`Fadvise::do_fadvise64(fd, offset, len, advice) -> isize`:
1. let file = fdget(fd).ok_or(EBADF)?;
2. if file.is_o_path() { return -EBADF; }
3. if offset < 0 || len < 0 { return -EINVAL; }
4. let end = match offset.checked_add(len) {
5.   Some(e) => e,
6.   None    => return -EINVAL,
7. };
8. let inode = file.inode();
9. if inode.is_pipe() || inode.is_socket() || inode.is_fifo() { return -ESPIPE; }
10. if inode.is_dir() { return -EINVAL; }
11. if !inode.is_regular() && !inode.is_blkdev() { return -EINVAL; }
12. if let Some(fadvise_hook) = inode.fops().fadvise {
13.   return fadvise_hook(&file, offset, len, advice);
14. }
15. Fadvise::generic_fadvise(&file, offset, len, advice)

`Fadvise::generic_fadvise(file, offset, len, advice) -> isize`:
1. let mapping = file.mapping();
2. let bdi = mapping.backing_dev_info();
3. let actual_len = if len == 0 { i_size_read(mapping.host) - offset } else { len };
4. match advice {
5.   POSIX_FADV_NORMAL => {
6.     file.f_ra.ra_pages.store(bdi.ra_pages, Relaxed);
7.   }
8.   POSIX_FADV_RANDOM => {
9.     file.f_ra.ra_pages.store(0, Relaxed);
10.  }
11.  POSIX_FADV_SEQUENTIAL => {
12.    file.f_ra.ra_pages.store(bdi.ra_pages * 2, Relaxed);
13.  }
14.  POSIX_FADV_WILLNEED => {
15.    let start_pg = offset >> PAGE_SHIFT;
16.    let end_pg   = (offset + actual_len + PAGE_SIZE - 1) >> PAGE_SHIFT;
17.    force_page_cache_readahead(mapping, file, start_pg, end_pg - start_pg);
18.  }
19.  POSIX_FADV_DONTNEED => {
20.    let start = round_up(offset, PAGE_SIZE);
21.    let end   = round_down(offset + actual_len, PAGE_SIZE);
22.    if end > start {
23.      invalidate_mapping_pages(mapping, start >> PAGE_SHIFT, (end >> PAGE_SHIFT) - 1);
24.    }
25.  }
26.  POSIX_FADV_NOREUSE => { /* no-op */ }
27.  _ => return -EINVAL,
28. }
29. 0

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_validated` | INVARIANT | EBADF returned before any work on closed fd. |
| `offset_len_non_negative` | INVARIANT | offset, len ≥ 0. |
| `overflow_checked` | INVARIANT | offset + len overflow ⟹ EINVAL. |
| `fd_type_validated` | INVARIANT | pipe/socket/fifo/dir/anon ⟹ EINVAL/ESPIPE. |
| `advice_total` | INVARIANT | only six POSIX_FADV_* accepted, others ⟹ EINVAL. |
| `dontneed_clean_only` | INVARIANT | DONTNEED never frees dirty pages. |

### Layer 2: TLA+

`mm/fadvise.tla`:
- States: per-fdget, per-validate-range, per-advice-dispatch, per-readahead, per-invalidate.
- Properties:
  - `safety_invalidate_clean_only`.
  - `safety_overflow_rejected`.
  - `safety_advice_total`.
  - `safety_willneed_bounded_by_memory`.
  - `liveness_advice_returns_zero_or_error`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_fadvise64` post: ret 0 ⟹ advice applied per advice table | `Fadvise::do_fadvise64` |
| `generic_fadvise` post: ra_pages updated for NORMAL/RANDOM/SEQUENTIAL | `Fadvise::generic_fadvise` |
| `force_page_cache_readahead` post: ≤ bdi.ra_pages prefetched per slot | `Fadvise::generic_fadvise` |
| `invalidate_mapping_pages` post: clean pages dropped | `Fadvise::generic_fadvise` |

### Layer 4: Verus / Creusot functional

Per-`posix_fadvise(2)` / `fadvise64(2)` man-page and per-`mm/fadvise.c` semantic equivalence. LTP `fadvise01..03` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fadvise64(2)` reinforcement:

- **Per-fd strict EBADF** — defense against per-invalid-fd kernel work.
- **Per-range overflow check** — defense against per-offset+len arithmetic overflow.
- **Per-advice total dispatch** — defense against per-unknown-advice undefined behavior.
- **Per-DONTNEED clean-only** — defense against per-data-loss on dirty range.
- **Per-WILLNEED memory-pressure bound** — defense against per-prefetch-OOM.

## Grsecurity / PaX surface

- **PaX UDEREF none applicable** — fadvise64 takes no user pointers; the args are integer fd/offset/len/advice. No SMAP-relevant copy_from_user.
- **fadvise EBADF strict** — under hardened mode, fdget validation is strict: closed fd, O_PATH fd, anon-inode fd without `fadvise` fops, and stale per-pidns fds all return `-EBADF` immediately; defense against per-fd-confusion downstream.
- **CAP_SYS_ADMIN gate on cross-uid DONTNEED** — under GRKERNSEC_FADVISE_HARDEN, a non-owner caller invoking POSIX_FADV_DONTNEED on a shared file requires `CAP_SYS_ADMIN`. Defense against per-cache-eviction DoS where one tenant evicts another's hot pages.
- **GRKERNSEC_TPE-correlation** — Trusted-Path Execution restricts fadvise on TPE-owned binaries: DONTNEED on an executable's text mapping requires `CAP_SYS_ADMIN`. Defense against per-text-cache-flush-induced-page-fault timing attack.
- **GRKERNSEC_HIDESYM on bdi.ra_pages exposure** — readahead settings inspected via fadvise side-channel (timing) gated; under hardened mode the readahead-window-change is bounded by per-uid limits.
- **GRKERNSEC_FADVISE_RATE_LIMIT** — fadvise calls per-uid rate-limited (default 10k/sec); spam triggers audit + throttle. Defense against per-fadvise-induced cache thrashing DoS.
- **PAX_REFCOUNT on file refcount** — defense against per-refcount-overflow UAF on parallel fadvise + close race.
- **GRKERNSEC_AUDIT_FADVISE_DONTNEED** — POSIX_FADV_DONTNEED on files outside the caller's uid logged with PID, comm, path, offset, len. Used to detect cross-tenant cache attacks.
- **kernel_lockdown integrity-mode** — fadvise unchanged under lockdown (advice is integrity-preserving), but POSIX_FADV_DONTNEED on the kernel-image bdev or on integrity-measured files is rejected with `-EPERM`. Defense against per-IMA-cache invalidation.
- **GRKERNSEC_KSTACKOVERFLOW** — defense against per-fadvise-deep-callchain stack overflow (especially in fs-specific fadvise hooks).
- **fs-fadvise hook validation** — under hardened mode, only built-in fs `fadvise` hooks are accepted; out-of-tree modules cannot register custom hooks.
- **PAX_USERCOPY_HARDEN not directly relevant** — no userspace copy on this path.
- **GRKERNSEC_BPF_HARDEN correlation** — fadvise tracepoints (`syscalls:sys_enter_fadvise64`) gated by `CAP_PERFMON` + perf_paranoid level under hardened mode.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Readahead window algorithm (covered in Tier-3 `mm/readahead.md`).
- Page-cache invalidation internals (covered in Tier-3 `mm/truncate.md`).
- fs-specific fadvise hooks (covered in per-fs Tier-3 docs).
- Implementation code.
