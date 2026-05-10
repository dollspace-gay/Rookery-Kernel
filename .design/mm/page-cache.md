# Tier-3: mm/page-cache — filemap, folio operations, readahead, truncate

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - mm/filemap.c
  - mm/folio-compat.c
  - mm/readahead.c
  - mm/truncate.c
  - mm/util.c
  - include/linux/pagemap.h
  - include/linux/page-flags.h
  - include/linux/folio_queue.h
-->

## Summary
Tier-3 design for the kernel's page cache: file-backed and shmem-backed pages keyed by `(struct address_space, pgoff_t)`. Owns filemap (the cache's xarray-keyed lookup + insert + invalidate), folio operations (the cache's data unit), readahead heuristics + window management, and the truncate path. Consumes `mm/page-allocator.md` for the underlying physical pages and `lib/data-structures.md` xarray for the keyed lookup.

The page cache is the second-largest single user of memory in a typical kernel (after slab); it backs every read/write to file-backed pages, every mmap of a regular file, every `splice`/`sendfile`. Filesystems plug into it via `address_space_operations`.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Page-cache core | `mm/filemap.c` |
| Folio compat helpers | `mm/folio-compat.c` |
| Readahead heuristics | `mm/readahead.c` |
| Truncate (file-shrink) | `mm/truncate.c` |
| mm util helpers | `mm/util.c` (selected functions for page-cache) |
| Public API | `include/linux/pagemap.h`, `include/linux/page-flags.h`, `include/linux/folio_queue.h` |

## Compatibility contract

### `address_space_operations` vtable

`include/linux/fs.h` defines `struct address_space_operations`. Filesystems plug in their `read_folio`, `writepage`, `dirty_folio`, `release_folio`, `invalidate_folio`, `migrate_folio`, etc. handlers. **Function-pointer table layout byte-identical** so existing fs/* implementations work without recompilation against Rookery's kernel.

### Page-cache xarray

`address_space->i_pages` is an xarray keyed by `pgoff_t` (page offset within file). The xarray's MARK_PAGECACHE_DIRTY, MARK_PAGECACHE_TOWRITE, MARK_PAGECACHE_WRITEBACK marks visible to userspace via:
- `/proc/<pid>/pagemap` per-page bits
- `fadvise(POSIX_FADV_DONTNEED)` invalidation semantics
- `sync_file_range(2)` syscall

Identical to upstream.

### Folio operations

`struct folio` is the page-cache data unit. Operations:
- `folio_lock` / `folio_unlock` (exclusive lock; PageLocked bit)
- `folio_wait_locked` / `folio_wait_dirty`
- `folio_mark_dirty` (xarray dirty mark + per-folio Dirty bit + per-mm dirty accounting)
- `folio_clear_dirty_for_io`
- `folio_start_writeback` / `folio_end_writeback`
- `folio_get` / `folio_put` (saturating refcount)
- `folio_reference_held` (mapped count)

Behavior byte-identical so filesystem implementations work unmodified.

### Readahead window

`file->f_ra` (`struct file_ra_state`) holds per-file readahead window: start, size, async_size, prev_pos, ra_pages. The heuristics (sequential vs. random detection, window expansion + retreat, async readahead trigger) match upstream so fadvise + readahead selftests pass.

### Userspace-visible interfaces

- `posix_fadvise(2)` — readahead/dontneed/willneed/etc. semantics identical
- `madvise(MADV_WILLNEED|MADV_DONTNEED)` — same
- `readahead(2)` syscall — identical
- `sync_file_range(2)` — page-writeback flush identical
- `fadvise(POSIX_FADV_DONTNEED)` — invalidate clean pages
- `sync(2)` / `syncfs(2)` / `fsync(2)` / `fdatasync(2)` — flush dirty pages

### `/proc/meminfo` page-cache fields

- `Cached` — page cache size in pages
- `Buffers` — buffer cache (legacy block-FS) size
- `Dirty` — dirty pages awaiting writeback
- `Writeback` — pages currently in writeback
- `WritebackTmp` — writeback temp accounting

Format-identical.

## Requirements

- REQ-1: `address_space->i_pages` xarray maintains pgoff-keyed folio mapping; lookup/insert/erase byte-identical to upstream.
- REQ-2: `address_space_operations` function-pointer-table layout byte-identical so existing fs/* drivers + out-of-tree implementations work unchanged (recompile-from-source compat per `00-overview.md` REQ-4 / D2).
- REQ-3: Folio lifecycle (allocate → insert into cache → mark dirty → writeback → invalidate / truncate → free) preserves upstream's per-state invariants:
  - Newly-allocated folio: not in any cache, refcount = 1
  - Cache-resident folio: in `i_pages` xarray, refcount ≥ 1
  - Locked folio: PageLocked set; unique writer; readers may still hold references
  - Dirty folio: PageDirty set + xarray DIRTY mark + per-mm dirty accounting incremented
  - Writeback-in-progress: PageWriteback set + xarray TOWRITE/WRITEBACK marks
  - Invalidated folio: not in cache, but may be still referenced by readers (handled via release_folio path)
- REQ-4: Folio operations (`folio_lock`, `folio_mark_dirty`, etc.) match upstream byte-for-byte; lockdep ordering identical.
- REQ-5: Readahead heuristics: per-file window grows on sequential access, retreats on random access; async readahead triggers at `ra_pages / 2` boundary; detection threshold matches upstream (visible via tracepoints + dmesg).
- REQ-6: Truncate path (`truncate_inode_pages_range` family) handles in-flight readers correctly: a page being truncated while a reader holds it gets RELEASE-d after the reader finishes, never freed-while-mapped.
- REQ-7: `posix_fadvise`, `readahead(2)`, `sync_file_range(2)` syscalls byte-identical entry/exit + observable effects.
- REQ-8: `/proc/meminfo` page-cache fields byte-identical.
- REQ-9: shmem (tmpfs) integration: `shmem_aops` operations table populated; tmpfs files get the same page-cache treatment as disk-backed files.
- REQ-10: DAX integration: DAX-backed mappings bypass the page cache via `vm_ops->huge_fault` returning DAX-PFN directly; cross-ref `fs/00-overview.md` § direct-io-dax.md.
- REQ-11: Memcg integration: per-folio `folio->memcg_data` tracks the charging memcg; `__lruvec_stat_mod_folio` updates the cgroup-LRU stats. Cross-ref `mm/memcg.md` (Tier-3 to be authored).
- REQ-12: KASAN/KMSAN/KFENCE compatibility: page-cache operations transparently report sanitizer events.
- REQ-13: Layer-3 invariant harnesses on the i_pages xarray inherited from `lib/data-structures.md` § xarray invariants.
- REQ-14: TLA+ model `models/mm/page_cache_xarray.tla` proves concurrent reader/writer ordering: writers (filemap_add_folio) and readers (filemap_get_folio) coordinate via xarray's spinlock + RCU.
- REQ-15: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A `pahole struct address_space_operations` produces byte-identical layout vs. upstream. (covers REQ-2)
- [ ] AC-2: `mm/filemap_test.c`-style harness exercising filemap_get_folio, filemap_add_folio, filemap_remove_folio under concurrent reader/writer threads passes without xarray-corruption assertions. (covers REQ-1, REQ-13, REQ-14)
- [ ] AC-3: A folio lifecycle test (`tools/testing/selftests/mm/page-cache-state-machine`) walks every state transition and verifies invariants per the table in REQ-3. (covers REQ-3, REQ-4)
- [ ] AC-4: A sequential-read benchmark (`fio --ioengine=mmap --bs=4k`) on a 1 GiB file shows readahead-window growth tracking upstream's window-size telemetry within ±10% per request. (covers REQ-5)
- [ ] AC-5: A truncate test that races `truncate(file, 0)` with a concurrent `read(fd, ...)` produces no UAF / no read-past-EOF. (covers REQ-6)
- [ ] AC-6: A `posix_fadvise(POSIX_FADV_DONTNEED)` test invalidates the file's clean pages; subsequent read repopulates from disk. (covers REQ-7)
- [ ] AC-7: `cat /proc/meminfo | grep -E '^(Cached|Dirty|Writeback)'` byte-identical (modulo dynamic counts) on equivalent workload. (covers REQ-8)
- [ ] AC-8: A tmpfs file under heavy I/O exhibits the same page-cache behavior (Cached count, dirty marks) as a disk-backed file. (covers REQ-9)
- [ ] AC-9: A DAX-backed mmap (on a pmem device) shows zero Cached growth in `/proc/meminfo` (DAX bypasses page cache). (covers REQ-10)
- [ ] AC-10: A memcg-confined process's writes to a file are charged to its memcg; visible via `cat /sys/fs/cgroup/<grp>/memory.stat | grep 'file '`. (covers REQ-11)
- [ ] AC-11: `make verify` passes `kani::proofs::mm::page_cache::*` Kani harnesses. (covers REQ-13)
- [ ] AC-12: `make tla` passes `models/mm/page_cache_xarray.tla`. (covers REQ-14)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-15)

## Architecture

### Rust module organization

- `kernel::mm::page_cache::filemap` — i_pages xarray + folio insert/lookup/erase
- `kernel::mm::page_cache::folio_ops` — folio lock/unlock/mark_dirty/start_writeback
- `kernel::mm::page_cache::readahead` — per-file readahead window
- `kernel::mm::page_cache::truncate` — file-shrink + invalidate path
- `kernel::mm::page_cache::aops::AddressSpaceOps` — address_space_operations vtable trait
- `kernel::mm::folio::Folio` — folio type (extended from `mm/page-allocator.md`)

### Locking and concurrency

- **`address_space->i_pages` xarray lock**: spinlock; held during xarray mutation. Readers use `xa_load` under RCU.
- **Per-folio `folio->_lock`** (PageLocked bit): atomic; bit-spin lock. Held for exclusive folio access (during read-folio fault, during writeback).
- **`address_space->i_mmap_rwsem`**: protects the address_space's reverse-map (cross-ref `mm/virtual-memory.md`).
- **`address_space->private_lock`** (per-fs spinlock): used by some fs (e.g., bufferhead-using ext4 for buffer-list manipulation).
- **Per-mm dirty accounting**: cgroup writeback + per-mm `pinned_vm` etc. updated under `mm->page_table_lock` via `__lruvec_stat_mod_folio`.

### Error handling

- `Err(ENOMEM)` — folio allocation failed (consumed page-allocator returned ENOMEM)
- `Err(EAGAIN)` — folio truncated mid-read; caller retries
- `Err(EIO)` — read-folio backend returned I/O error
- `Err(ENOSPC)` — write to file would exceed size; flusher consumed the error
- `Err(ESTALE)` — NFS-style stale handle; folio invalidated

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| filemap xarray insert (under xa_lock) | `kani::proofs::mm::page_cache::xa_insert_safety` |
| filemap_get_folio under RCU read-side | `kani::proofs::mm::page_cache::xa_lookup_safety` |
| folio_lock bit-spin | `kani::proofs::mm::page_cache::folio_lock_safety` |
| Truncate-while-reader page release | `kani::proofs::mm::page_cache::truncate_release_safety` |
| Readahead window expansion arithmetic | `kani::proofs::mm::page_cache::readahead_arith_safety` |

### Layer 2: TLA+ models

- `models/mm/page_cache_xarray.tla` (NEW) — proves concurrent filemap_add_folio + filemap_get_folio + filemap_remove_folio coordinate correctly via xarray's spinlock + RCU. Liveness: every reader either observes a coherent snapshot or retries.
- `models/mm/folio_writeback.tla` (NEW) — proves folio_start_writeback + folio_end_writeback ordering: a folio enters writeback exactly once per dirty cycle; the dirty bit and writeback bit do not coexist.

### Layer 3: Kani harnesses for data-structure invariants

- xarray invariants (inherited from `lib/data-structures.md` § xarray) — enforced by the underlying lib structure
- folio refcount + pin count (inherited from `mm/00-overview.md` § folio refcount-with-pins invariant)
- address_space mapping consistency: `kani::proofs::mm::page_cache::aops_consistency_invariant` — every folio in `i_pages` has `folio->mapping` == the address_space; every folio in `i_pages` has the address_space's a_ops vtable

### Layer 4: Functional correctness (opt-in)

- **Readahead window adaptation** via Creusot — proves: under sequential access, the window monotonically grows (until cap); under random access, the window shrinks (until floor). Tractable; relevant for fadvise tests.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MEMORY_SANITIZE** (zero on free) | Folios freed from page cache (truncate / invalidate) get zeroed unless explicitly opt-out via per-cache `SLAB_NO_ZERO_ON_FREE` (folios use the page allocator, so the page-allocator default applies — `vm.zero_on_free=1`) | § Default-on configurable off |
| **AUTOSLAB** | folio allocations route through `KmemCache::<Folio>::new()` per-type; the underlying page from page-allocator is per-page accounted | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT**: folio->_refcount uses `Refcount` (saturating)
- **UDEREF**: never accepts user pointers directly; readers/writers go through `iov_iter` (cross-ref `lib/usercopy.md`)
- **SIZE_OVERFLOW**: pgoff_t arithmetic uses checked operators
- **CONSTIFY**: address_space_operations vtables provided by filesystems are `static const`

### Row-2 / GR-RBAC integration

LSM hooks fire from filesystem read/write paths (cross-ref individual fs/* docs), not from this component directly. The page cache is infrastructure.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — page-cache semantics are exhaustively specified by VFS contract + Linux folio extensions)

## Out of Scope

- Per-filesystem read_folio implementations (cross-ref individual fs/ Tier-3 docs)
- DAX implementation (cross-ref `fs/00-overview.md` § direct-io-dax.md)
- Memcg charging algorithm (cross-ref `mm/memcg.md`)
- 32-bit-only paths
- Implementation code
