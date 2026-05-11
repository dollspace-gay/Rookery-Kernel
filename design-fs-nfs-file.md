---
title: "Tier-3: fs/nfs/file.c — NFS regular-file operations"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-11
updated: 2026-05-11
---


## Design Specification

### Summary

NFS regular-file operations bridge the kernel VFS to the NFS RPC protocol family. Per-`nfs_file_operations` exposes the file `read_iter` / `write_iter` / `mmap_prepare` / `open` / `flush` / `release` / `fsync` / `lock` / `flock` / `splice_read` / `splice_write` / `check_flags` set; per-`nfs_file_aops` provides the address-space side (read_folio, readahead, writepages, write_begin/write_end, invalidate_folio, release_folio, migrate_folio, launder_folio, is_dirty_writeback, swap_activate/deactivate/swap_rw). Per-`nfs_file_open` revalidates the open context via `nfs_open` (handles NFS_INO_INVALID_DATA refresh on the inode); per-`nfs_file_release` clears the open context and releases fscache state. Per-`nfs_file_read` chooses between `nfs_file_direct_read` (IOCB_DIRECT) and cached path: takes the per-inode read-IO serializer (`nfs_start_io_read`), revalidates the mapping, then defers to `generic_file_read_iter` over the page cache. Per-`nfs_file_write` mirrors that: per-IOCB_DIRECT → `nfs_file_direct_write`; otherwise revalidate-size if O_APPEND or beyond-EOF, take the per-inode write serializer (`nfs_start_io_write`), invoke `generic_write_checks` + `generic_perform_write` (which calls `nfs_write_begin` / `nfs_write_end` over the page cache and stages NFS-private dirty page tracking via `nfs_update_folio` for `nfs_writepages` / `nfs_write_pageio` to commit later). Per-`nfs_file_fsync` loops `file_write_and_wait_range` + `nfs_commit_inode(FLUSH_SYNC)` + `pnfs_sync_inode` until per-`redirtied_pages` counter is stable (handles re-dirtying during commit). Per-`nfs_file_mmap_prepare` revalidates the mapping and installs `nfs_file_vm_ops` (filemap_fault, filemap_map_pages, per-`nfs_vm_page_mkwrite` for COW-on-shared-write). Per-write-delegation (NFSv4): allows local-fs-like buffered writes without per-write OTW round-trip; per-`do_unlk` / `do_setlk` checks `nfs_have_read_or_write_delegation` to short-circuit lock/conflict tests. Per-pNFS layout integration is via `pnfs_ld_read_whole_page` (in write_begin RMW heuristic) and `pnfs_sync_inode` (in fsync). Critical for: NFS-cached I/O correctness, NFS close-to-open semantics, NFS commit-protocol UNSTABLE→STABLE staging, NFS-mmap consistency.

This Tier-3 covers `fs/nfs/file.c` (~968 lines).

### Acceptance Criteria

- [ ] AC-1: nfs_check_flags rejects O_APPEND | O_DIRECT with -EINVAL.
- [ ] AC-2: nfs_file_open sets FMODE_CAN_ODIRECT on success.
- [ ] AC-3: nfs_file_read with IOCB_DIRECT bypasses page cache via nfs_file_direct_read.
- [ ] AC-4: nfs_file_read (buffered) revalidates the mapping before generic_file_read_iter.
- [ ] AC-5: nfs_file_write with O_APPEND or beyond-EOF triggers nfs_revalidate_file_size.
- [ ] AC-6: nfs_file_write on a swap-file returns -ETXTBSY.
- [ ] AC-7: nfs_file_flush on FMODE_WRITE flushes wb_all and reports wb_err.
- [ ] AC-8: nfs_file_fsync loops until redirtied_pages counter is stable.
- [ ] AC-9: nfs_file_fsync invokes pnfs_sync_inode after nfs_commit_inode.
- [ ] AC-10: nfs_file_mmap_prepare installs nfs_file_vm_ops and revalidates mapping.
- [ ] AC-11: nfs_vm_page_mkwrite returns VM_FAULT_LOCKED on successful update, VM_FAULT_SIGBUS on incompatible-flush failure.
- [ ] AC-12: nfs_write_begin RMW when partial-write to non-uptodate folio and FMODE_READ.
- [ ] AC-13: nfs_release_folio refuses release under kswapd/kcompactd when folio is private.
- [ ] AC-14: nfs_check_dirty_writeback marks writeback when commit_info.rpcs_out > 0.
- [ ] AC-15: nfs_lock with FL_RECLAIM returns -ENOGRACE.
- [ ] AC-16: do_setlk after server lock zaps caches unless delegation present.
- [ ] AC-17: nfs_swap_activate with holes (blocks*512 < isize) returns -EINVAL.

### Architecture

```
struct NfsFileOperations {  // const, registered with VFS
  llseek:      Nfs::file_llseek,
  read_iter:   Nfs::file_read,
  write_iter:  Nfs::file_write,
  mmap_prepare: Nfs::file_mmap_prepare,
  open:        Nfs::file_open,
  flush:       Nfs::file_flush,
  release:     Nfs::file_release,
  fsync:       Nfs::file_fsync,
  lock:        Nfs::lock,
  flock:       Nfs::flock,
  splice_read: Nfs::file_splice_read,
  splice_write: iter_file_splice_write,
  check_flags: Nfs::check_flags,
  fop_flags:   FOP_DONTCACHE,
}

struct NfsFileAops {
  read_folio:        Nfs::read_folio,
  readahead:         Nfs::readahead,
  dirty_folio:       filemap_dirty_folio,
  writepages:        Nfs::writepages,           // via nfs_write_pageio
  write_begin:       NfsAops::write_begin,
  write_end:         NfsAops::write_end,
  invalidate_folio:  NfsAops::invalidate_folio,
  release_folio:     NfsAops::release_folio,
  migrate_folio:     Nfs::migrate_folio,
  launder_folio:     NfsAops::launder_folio,
  is_dirty_writeback: NfsAops::check_dirty_writeback,
  error_remove_folio: generic_error_remove_folio,
  swap_activate:     NfsAops::swap_activate,
  swap_deactivate:   NfsAops::swap_deactivate,
  swap_rw:           Nfs::swap_rw,
}

struct NfsFileVmOps {
  fault:       filemap_fault,
  map_pages:   filemap_map_pages,
  page_mkwrite: NfsFileVm::page_mkwrite,
}
```

`NfsFile::open(inode, filp) -> Result<()>`:
1. nfs_inc_stats(inode, NFSIOS_VFSOPEN).
2. Self::check_flags(filp.f_flags)?.
3. nfs_open(inode, filp)?  /* allocates nfs_open_context, may force NFS_INO_INVALID_DATA refresh */
4. filp.f_mode |= FMODE_CAN_ODIRECT.

`NfsFile::release(inode, filp) -> 0`:
1. nfs_inc_stats(NFSIOS_VFSRELEASE).
2. nfs_file_clear_open_context(filp).
3. nfs_fscache_release_file(inode, filp).

`NfsFile::read(iocb, to) -> ssize_t`:
1. if iocb.ki_flags & IOCB_DIRECT: return Self::file_direct_read(iocb, to, false).
2. nfs_start_io_read(inode)?  /* per-inode read serialization vs writes */
3. r = nfs_revalidate_mapping(inode, mapping).
4. if r == 0:
   - r = generic_file_read_iter(iocb, to).
   - if r > 0: nfs_add_stats(NFSIOS_NORMALREADBYTES, r).
5. nfs_end_io_read(inode).
6. return r.

`NfsFile::write(iocb, from) -> ssize_t`:
1. nfs_key_timeout_notify(file, inode)?.
2. if iocb.ki_flags & IOCB_DIRECT: return Self::file_direct_write(...).
3. if IS_SWAPFILE(inode): return -ETXTBSY.
4. if iocb.ki_flags & IOCB_APPEND ∨ ki_pos > i_size_read: Self::revalidate_size?.
5. nfs_clear_invalid_mapping(mapping).
6. since = filemap_sample_wb_err.
7. nfs_start_io_write(inode)?.
8. r = generic_write_checks(iocb, from).
9. if r > 0: r = generic_perform_write(iocb, from).
10. nfs_end_io_write(inode).
11. if r > 0:
    - nfs_add_stats(NFSIOS_NORMALWRITTENBYTES, r).
    - if NFS_MOUNT_WRITE_EAGER: filemap_fdatawrite_range(ki_pos - r, ki_pos - 1).
    - if NFS_MOUNT_WRITE_WAIT: filemap_fdatawait_range(...).
    - r = generic_write_sync(iocb, r); if r < 0: return r.
12. err = filemap_check_wb_err.
13. match err { EDQUOT | EFBIG | ENOSPC => nfs_wb_all(inode); err' = file_check_and_advance_wb_err. }
14. return r.

`NfsFile::fsync(file, start, end, datasync) -> i32`:
1. save = NFS_I(inode).redirtied_pages.
2. loop:
   - r = file_write_and_wait_range(file, start, end)?; break-on-err.
   - r = Self::fsync_commit(file, datasync)?; break-on-err.
   - r = pnfs_sync_inode(inode, datasync)?; break-on-err.
   - n = NFS_I(inode).redirtied_pages.
   - if n == save: break.
   - save = n.
3. return r.

`NfsFile::fsync_commit(file, datasync) -> i32`:
1. nfs_inc_stats(NFSIOS_VFSFSYNC).
2. r = nfs_commit_inode(inode, FLUSH_SYNC).  /* NFS COMMIT RPC */
3. r2 = file_check_and_advance_wb_err(file).
4. return if r2 < 0 { r2 } else { r }.

`NfsAops::write_begin(iocb, mapping, pos, len, &folio, &fsdata) -> i32`:
1. nfs_truncate_last_folio(mapping, i_size_read(host), pos).
2. start: folio = write_begin_get_folio(iocb, mapping, pos>>PAGE_SHIFT, len)?.
3. *foliop = folio.
4. r = nfs_flush_incompatible(file, folio).
5. if r != 0: folio_unlock; folio_put; return r.
6. if !once_thru ∧ Self::want_rmw(file, folio, pos, len):
   - once_thru = 1.
   - folio_clear_dropbehind(folio).
   - r = nfs_read_folio(file, folio); folio_put.
   - if r == 0: goto start.
7. return r.

`NfsAops::write_end(iocb, mapping, pos, len, copied, folio, fsdata) -> i32`:
1. if !folio_test_uptodate(folio): zero-fill (head / tail / cross-pglen).
2. status = nfs_update_folio(file, folio, offset, copied).
3. folio_unlock; folio_put.
4. if status < 0: return status.
5. NFS_I(host).write_io += copied.
6. if nfs_ctx_key_to_expire(ctx, host): nfs_wb_all(host).
7. return copied.

`NfsAops::release_folio(folio, gfp) -> bool`:
1. if folio_test_private(folio):
   - if !(current_gfp_context(gfp) & GFP_KERNEL) ∨ kswapd ∨ kcompactd: return false.
   - if nfs_wb_folio_reclaim < 0 ∨ still-private: return false.
2. return nfs_fscache_release_folio(folio, gfp).

`NfsFileVm::page_mkwrite(vmf) -> vm_fault_t`:
1. sb_start_pagefault(inode.i_sb).
2. if folio_test_private_2 ∧ folio_wait_private_2_killable < 0: return VM_FAULT_RETRY.
3. wait_on_bit_action(NFS_INO_INVALIDATING).
4. folio_lock.
5. if folio.mapping != inode.i_mapping: ret = VM_FAULT_NOPAGE; goto out_unlock.
6. folio_wait_writeback.
7. if nfs_folio_length(folio) == 0: ret = VM_FAULT_NOPAGE; goto out_unlock.
8. if nfs_flush_incompatible == 0 ∧ nfs_update_folio == 0: ret = VM_FAULT_LOCKED; goto out.
9. else: ret = VM_FAULT_SIGBUS.

`NfsFile::lock(filp, cmd, fl) -> i32`:
1. if fl.flc_flags & FL_RECLAIM: return -ENOGRACE.
2. is_local = NFS_SERVER.flags & NFS_MOUNT_LOCAL_FCNTL.
3. if NFS_PROTO.lock_check_bounds: r = NFS_PROTO.lock_check_bounds(fl)?.
4. match cmd { IS_GETLK => do_getlk, lock_is_unlock => do_unlk, else => do_setlk }.

### Out of Scope

- fs/nfs/direct.c direct-IO bypass (`nfs_file_direct_read` / `_write`) (covered in `direct.md` Tier-3)
- fs/nfs/write.c writeback / commit machinery (covered in `write.md` Tier-3)
- fs/nfs/read.c readahead / read_folio internals (covered in `read.md` Tier-3)
- fs/nfs/delegation.c NFSv4 delegation lifecycle (covered in `delegation.md` Tier-3)
- fs/nfs/pnfs.c pNFS layout state-machine (covered in `pnfs.md` Tier-3)
- fs/nfs/nfs4file.c NFSv4 file-ops (clone / copy_file_range / dedup) (covered in `nfs4file.md` Tier-3)
- fs/nfs/nfs4proc.c / nfs3proc.c RPC ops (covered separately)
- fs/nfs/inode.c inode/cache lifecycle (covered in `inode.md` Tier-3)
- fs/nfs/dir.c directory operations (covered in `dir.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct file_operations nfs_file_operations` | per-VFS file-op table | `NfsFileOperations` |
| `struct address_space_operations nfs_file_aops` | per-mapping op table | `NfsFileAops` |
| `struct vm_operations_struct nfs_file_vm_ops` | per-VMA op table | `NfsFileVmOps` |
| `nfs_check_flags()` | per-open flag check (O_APPEND ∧ O_DIRECT ⟹ EINVAL) | `Nfs::check_flags` |
| `nfs_file_open()` | per-open: stats + check_flags + nfs_open + FMODE_CAN_ODIRECT | `NfsFile::open` |
| `nfs_file_release()` | per-close: clear ctx + fscache release | `NfsFile::release` |
| `nfs_revalidate_file_size()` | per-O_DIRECT/invalid-size forced revalidate | `NfsFile::revalidate_size` |
| `nfs_file_llseek()` | per-llseek; SEEK_END/DATA/HOLE forces size-revalidate | `NfsFile::llseek` |
| `nfs_file_flush()` | per-flush: wb_all + check_wb_err (close-to-open write-back) | `NfsFile::flush` |
| `nfs_file_read()` | per-read_iter: direct or generic page-cache | `NfsFile::read` |
| `nfs_file_splice_read()` | per-splice-read | `NfsFile::splice_read` |
| `nfs_file_mmap_prepare()` | per-mmap-prep: revalidate + install nfs_file_vm_ops | `NfsFile::mmap_prepare` |
| `nfs_file_fsync_commit()` | per-fsync inner: nfs_commit_inode(FLUSH_SYNC) | `NfsFile::fsync_commit` |
| `nfs_file_fsync()` | per-fsync outer: loop until redirtied_pages stable | `NfsFile::fsync` |
| `nfs_truncate_last_folio()` | per-EOF-folio zero-tail on truncate | `NfsFile::truncate_last_folio` |
| `nfs_folio_is_full_write()` | per-write-begin: full-folio-write detection | `NfsFile::folio_is_full_write` |
| `nfs_want_read_modify_write()` | per-write-begin: RMW vs MWR heuristic | `NfsFile::want_rmw` |
| `nfs_write_begin()` | per-aops write_begin: lock folio + nfs_flush_incompatible + optional RMW read | `NfsAops::write_begin` |
| `nfs_write_end()` | per-aops write_end: zero-fill + nfs_update_folio + ctx-key-expire check | `NfsAops::write_end` |
| `nfs_invalidate_folio()` | per-aops invalidate: wb_folio or wb_folio_cancel | `NfsAops::invalidate_folio` |
| `nfs_release_folio()` | per-aops release: refuse if PG_private; else nfs_fscache_release_folio | `NfsAops::release_folio` |
| `nfs_check_dirty_writeback()` | per-aops dirty-writeback: rpcs_out ⟹ pretend writeback | `NfsAops::check_dirty_writeback` |
| `nfs_launder_folio()` | per-aops launder: wb_folio sync | `NfsAops::launder_folio` |
| `nfs_swap_activate()` | per-swap-activate: holes check + rpc_clnt_swap_activate | `NfsAops::swap_activate` |
| `nfs_swap_deactivate()` | per-swap-deactivate | `NfsAops::swap_deactivate` |
| `nfs_vm_page_mkwrite()` | per-VMA mkwrite: wb_folio_writeback + nfs_update_folio | `NfsFileVm::page_mkwrite` |
| `nfs_file_write()` | per-write_iter: direct or generic page-cache; per-WRITE_EAGER/WAIT mount-flags | `NfsFile::write` |
| `do_getlk()` / `do_unlk()` / `do_setlk()` | per-POSIX-lock helpers (delegation-aware) | `NfsFile::do_getlk` / `do_unlk` / `do_setlk` |
| `nfs_lock()` | per-POSIX fcntl lock | `NfsFile::lock` |
| `nfs_flock()` | per-flock(2) lock | `NfsFile::flock` |
| `nfs_have_read_or_write_delegation()` | per-NFSv4 delegation guard | `NfsDelegation::have_read_or_write` |
| `pnfs_ld_read_whole_page()` | per-pNFS layout-driver hint | `Pnfs::ld_read_whole_page` |
| `pnfs_sync_inode()` | per-pNFS layoutcommit | `Pnfs::sync_inode` |
| `nfs_commit_inode()` | per-NFS COMMIT RPC | `NfsCommit::commit_inode` |
| `nfs_wb_all()` / `nfs_wb_folio()` / `nfs_wb_folio_cancel()` / `nfs_wb_folio_reclaim()` | per-writeback drivers | `NfsWriteback::*` |
| `nfs_update_folio()` | per-folio dirty staging into nfs_page | `NfsWrite::update_folio` |
| `nfs_flush_incompatible()` | per-folio incompatible-owner flush | `NfsWrite::flush_incompatible` |

### compatibility contract

REQ-1: struct file_operations nfs_file_operations:
- llseek = nfs_file_llseek.
- read_iter = nfs_file_read.
- write_iter = nfs_file_write.
- mmap_prepare = nfs_file_mmap_prepare.
- open = nfs_file_open.
- flush = nfs_file_flush.
- release = nfs_file_release.
- fsync = nfs_file_fsync.
- lock = nfs_lock.
- flock = nfs_flock.
- splice_read = nfs_file_splice_read.
- splice_write = iter_file_splice_write.
- check_flags = nfs_check_flags.
- fop_flags = FOP_DONTCACHE.

REQ-2: nfs_check_flags(flags):
- if (flags & (O_APPEND | O_DIRECT)) == (O_APPEND | O_DIRECT): return -EINVAL.
- return 0.

REQ-3: nfs_file_open(inode, filp):
- nfs_inc_stats(inode, NFSIOS_VFSOPEN).
- res = nfs_check_flags(filp.f_flags); if res: return res.
- res = nfs_open(inode, filp). /* allocates nfs_open_context, refreshes inode (NFS_INO_INVALID_DATA path) per close-to-open */
- if res == 0: filp.f_mode |= FMODE_CAN_ODIRECT.
- return res.

REQ-4: nfs_file_release(inode, filp):
- nfs_inc_stats(inode, NFSIOS_VFSRELEASE).
- nfs_file_clear_open_context(filp).
- nfs_fscache_release_file(inode, filp).
- return 0.

REQ-5: nfs_revalidate_file_size(inode, filp):
- if filp.f_flags & O_DIRECT: goto force_reval.
- if nfs_check_cache_invalid(inode, NFS_INO_INVALID_SIZE): goto force_reval.
- return 0.
- force_reval: return __nfs_revalidate_inode(server, inode).

REQ-6: nfs_file_llseek(filp, offset, whence):
- if whence ∉ { SEEK_SET, SEEK_CUR }:
  - r = nfs_revalidate_file_size(inode, filp); if r<0 return r.
- return generic_file_llseek(filp, offset, whence).

REQ-7: nfs_file_flush(file, id):
- nfs_inc_stats(inode, NFSIOS_VFSFLUSH).
- if !(file.f_mode & FMODE_WRITE): return 0.
- since = filemap_sample_wb_err(file.f_mapping).
- nfs_wb_all(inode). /* flush + COMMIT path */
- return filemap_check_wb_err(file.f_mapping, since).

REQ-8: nfs_file_read(iocb, to):
- trace_nfs_file_read.
- if iocb.ki_flags & IOCB_DIRECT: return nfs_file_direct_read(iocb, to, false).
- r = nfs_start_io_read(inode); if r: return r.  /* per-inode read serializer */
- r = nfs_revalidate_mapping(inode, mapping); /* close-to-open or attr-cache-driven invalidate */
- if !r:
  - r = generic_file_read_iter(iocb, to).
  - if r>0: nfs_add_stats(NFSIOS_NORMALREADBYTES, r).
- nfs_end_io_read(inode).
- return r.

REQ-9: nfs_file_splice_read(in, ppos, pipe, len, flags):
- nfs_start_io_read, revalidate_mapping, filemap_splice_read, nfs_end_io_read; per-stats NFSIOS_NORMALREADBYTES.

REQ-10: nfs_file_mmap_prepare(desc):
- status = generic_file_mmap_prepare(desc).
- if !status:
  - desc.vm_ops = &nfs_file_vm_ops.
  - status = nfs_revalidate_mapping(inode, file.f_mapping).
- return status.

REQ-11: nfs_file_fsync_commit(file, datasync):
- nfs_inc_stats(NFSIOS_VFSFSYNC).
- ret = nfs_commit_inode(inode, FLUSH_SYNC). /* synchronous NFS COMMIT RPC */
- ret2 = file_check_and_advance_wb_err(file).
- if ret2 < 0: return ret2.
- return ret.

REQ-12: nfs_file_fsync(file, start, end, datasync):
- save_nredirtied = atomic_long_read(&NFS_I(inode).redirtied_pages).
- loop:
  - r = file_write_and_wait_range(file, start, end); if r: break.
  - r = nfs_file_fsync_commit(file, datasync); if r: break.
  - r = pnfs_sync_inode(inode, !!datasync); if r: break. /* pNFS layoutcommit */
  - n = atomic_long_read(&redirtied_pages); if n == save_nredirtied: break.
  - save_nredirtied = n.  /* loop again if redirty happened during commit */
- return r.

REQ-13: nfs_truncate_last_folio(mapping, from, to):
- if from >= to: return.
- folio = filemap_lock_folio(mapping, from >> PAGE_SHIFT); if IS_ERR: return.
- if folio_mkclean(folio): folio_mark_dirty(folio).
- if folio_test_uptodate(folio):
  - offset = from - folio_pos(folio).
  - end = min(folio_size(folio), to - folio_pos(folio)).
  - folio_zero_segment(folio, offset, end).
- folio_unlock; folio_put.

REQ-14: nfs_folio_is_full_write(folio, pos, len) → bool:
- pglen = nfs_folio_length(folio).
- offset = offset_in_folio(folio, pos).
- end = offset + len.
- return !pglen ∨ (end >= pglen ∧ !offset).

REQ-15: nfs_want_read_modify_write(file, folio, pos, len) → bool:
- if folio_test_uptodate ∨ folio_test_private ∨ nfs_folio_is_full_write: return false.
- if pnfs_ld_read_whole_page(inode): return true.
- if folio_test_dropbehind(folio): return false.
- if file.f_mode & FMODE_READ: return true.
- return false.

REQ-16: nfs_write_begin(iocb, mapping, pos, len, foliop, fsdata):
- nfs_truncate_last_folio(mapping, i_size_read(host), pos). /* zero past-EOF gap when extending */
- start: folio = write_begin_get_folio(iocb, mapping, pos>>PAGE_SHIFT, len).
- *foliop = folio.
- r = nfs_flush_incompatible(file, folio). /* per-incompatible open-context flush */
- if r: folio_unlock + folio_put.
- elif !once_thru ∧ nfs_want_read_modify_write:
  - once_thru = 1.
  - folio_clear_dropbehind(folio).
  - r = nfs_read_folio(file, folio); folio_put.
  - if !r: goto start.
- return r.

REQ-17: nfs_write_end(iocb, mapping, pos, len, copied, folio, fsdata):
- /* Zero uninitialised tail / head if file is being extended */
- if !folio_test_uptodate(folio):
  - pglen = nfs_folio_length(folio).
  - if pglen == 0: folio_zero_segments(0, offset, end, fsize); folio_mark_uptodate.
  - elif end >= pglen: folio_zero_segment(end, fsize); if offset == 0: folio_mark_uptodate.
  - else: folio_zero_segment(pglen, fsize).
- status = nfs_update_folio(file, folio, offset, copied). /* stage nfs_page on inode write-list */
- folio_unlock; folio_put.
- if status < 0: return status.
- NFS_I(host).write_io += copied.
- if nfs_ctx_key_to_expire(ctx, host): nfs_wb_all(host).
- return copied.

REQ-18: nfs_invalidate_folio(folio, offset, length):
- if offset != 0 ∨ length < folio_size: nfs_wb_folio(inode, folio).
- else: nfs_wb_folio_cancel(inode, folio).  /* full-folio invalidation cancels staged writes */
- folio_wait_private_2(folio). /* [DEPRECATED] fscache sync */

REQ-19: nfs_release_folio(folio, gfp) → bool:
- if folio_test_private(folio):
  - if !(current_gfp_context(gfp) & GFP_KERNEL) ∨ kswapd ∨ kcompactd: return false. /* avoid RPC under reclaim */
  - if nfs_wb_folio_reclaim(host, folio) < 0 ∨ still-private: return false.
- return nfs_fscache_release_folio(folio, gfp).

REQ-20: nfs_check_dirty_writeback(folio, *dirty, *writeback):
- if atomic_read(&NFS_I(host).commit_info.rpcs_out): *writeback = true; return.
- if folio_test_private(folio): *dirty = true.

REQ-21: nfs_launder_folio(folio):
- folio_wait_private_2(folio).
- return nfs_wb_folio(inode, folio).

REQ-22: nfs_swap_activate(sis, file, *span):
- if blocks*512 < isize: return -EINVAL ("swapfile has holes").
- rpc_clnt_swap_activate(clnt).
- add_swap_extent(sis, 0, sis.max, 0).
- *span = sis.pages.
- if cl.rpc_ops.enable_swap: cl.rpc_ops.enable_swap(inode).
- sis.flags |= SWP_FS_OPS.

REQ-23: nfs_swap_deactivate(file):
- rpc_clnt_swap_deactivate; cl.rpc_ops.disable_swap.

REQ-24: nfs_vm_page_mkwrite(vmf):
- sb_start_pagefault.
- folio_wait_private_2_killable; per-NFS_INO_INVALIDATING wait.
- folio_lock; if mapping != inode.i_mapping: unlock; return VM_FAULT_NOPAGE.
- folio_wait_writeback.
- if nfs_folio_length(folio) == 0: unlock; return NOPAGE.
- if nfs_flush_incompatible(filp, folio) == 0 ∧ nfs_update_folio(filp, folio, 0, pagelen) == 0: return VM_FAULT_LOCKED.
- else: return VM_FAULT_SIGBUS.

REQ-25: nfs_file_write(iocb, from):
- nfs_key_timeout_notify(file, inode); if r return r.
- if iocb.ki_flags & IOCB_DIRECT: return nfs_file_direct_write(iocb, from, false).
- if IS_SWAPFILE(inode): return -ETXTBSY.
- if iocb.ki_flags & IOCB_APPEND ∨ ki_pos > i_size_read(inode):
  - r = nfs_revalidate_file_size; if r return r.
- nfs_clear_invalid_mapping(file.f_mapping).
- since = filemap_sample_wb_err.
- nfs_start_io_write(inode).
- r = generic_write_checks(iocb, from). /* enforces ulimit, EFBIG, RLIMIT_FSIZE */
- if r > 0: r = generic_perform_write(iocb, from). /* calls aops.write_begin/end */
- nfs_end_io_write(inode).
- if r <= 0: goto out.
- written = r.
- nfs_add_stats(NFSIOS_NORMALWRITTENBYTES, written).
- if NFS_MOUNT_WRITE_EAGER: filemap_fdatawrite_range(ki_pos-written, ki_pos-1).
- if NFS_MOUNT_WRITE_WAIT: filemap_fdatawait_range(...).
- r = generic_write_sync(iocb, written); if r<0 return r.
- out: error = filemap_check_wb_err; per-EDQUOT/EFBIG/ENOSPC: nfs_wb_all + file_check_and_advance_wb_err.
- return r.

REQ-26: do_getlk(filp, cmd, fl, is_local):
- posix_test_lock(filp, fl).
- if fl.flc_type != F_UNLCK: conflict found; return.
- restore fl.flc_type.
- if nfs_have_read_or_write_delegation(inode): goto out_noconflict.
- if is_local: goto out_noconflict.
- return NFS_PROTO(inode).lock(filp, cmd, fl).

REQ-27: do_unlk(filp, cmd, fl, is_local):
- nfs_wb_all(inode). /* flush before changing lock state */
- l_ctx = nfs_get_lock_context(open_ctx).
- nfs_iocounter_wait(l_ctx). /* await in-flight I/O on this lock-owner */
- nfs_put_lock_context(l_ctx).
- if !is_local: NFS_PROTO.lock(filp, cmd, fl).
- else: locks_lock_file_wait(filp, fl).

REQ-28: do_setlk(filp, cmd, fl, is_local):
- nfs_sync_mapping(filp.f_mapping).
- if !is_local: NFS_PROTO.lock(filp, cmd, fl); else: locks_lock_file_wait.
- if r < 0: return.
- nfs_sync_mapping again.
- if !nfs_have_read_or_write_delegation(inode):
  - nfs_zap_caches(inode).
  - if mapping_mapped(filp.f_mapping): nfs_revalidate_mapping. /* lock acts as coherency point */

REQ-29: nfs_lock(filp, cmd, fl):
- if fl.flc_flags & FL_RECLAIM: return -ENOGRACE.
- is_local = (NFS_SERVER.flags & NFS_MOUNT_LOCAL_FCNTL) ? 1 : 0.
- lock_check_bounds (if present).
- per-IS_GETLK: do_getlk.
- per-lock_is_unlock: do_unlk.
- else: do_setlk.

REQ-30: nfs_flock(filp, cmd, fl):
- if !(fl.flc_flags & FL_FLOCK): return -ENOLCK.
- is_local = (NFS_MOUNT_LOCAL_FLOCK) ? 1 : 0.
- per-unlock: do_unlk; else do_setlk.

REQ-31: struct address_space_operations nfs_file_aops:
- read_folio, readahead, dirty_folio=filemap_dirty_folio, writepages=nfs_writepages, write_begin, write_end, invalidate_folio, release_folio, migrate_folio=nfs_migrate_folio, launder_folio, is_dirty_writeback, error_remove_folio=generic_error_remove_folio, swap_activate, swap_deactivate, swap_rw=nfs_swap_rw.

REQ-32: struct vm_operations_struct nfs_file_vm_ops:
- fault = filemap_fault; map_pages = filemap_map_pages; page_mkwrite = nfs_vm_page_mkwrite.

REQ-33: copy_file_range / clone_range / pNFS-layout:
- Not present in file.c (lives in fs/nfs/nfs4file.c for NFSv4 SSC and fs/nfs/pnfs.c). file.c integrates via pnfs_ld_read_whole_page (RMW heuristic) and pnfs_sync_inode (fsync layoutcommit).

REQ-34: NFSv4 write-delegation:
- Owned by fs/nfs/delegation.c. file.c queries via nfs_have_read_or_write_delegation to short-circuit lock RPCs and avoid post-unlock cache zap.

REQ-35: NFS commit protocol:
- writepage stages dirty page on inode commit_list (nfs_update_folio).
- nfs_commit_inode (called by flush/fsync) issues COMMIT RPC; on success transitions UNSTABLE→STABLE; otherwise re-dirties (redirtied_pages counter).
- fsync loops until redirtied_pages stable.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `check_flags_rejects_append_direct` | INVARIANT | per-check_flags: (O_APPEND | O_DIRECT) ⟹ -EINVAL. |
| `write_iocb_direct_bypasses_pgcache` | INVARIANT | per-file_write: IOCB_DIRECT ⟹ nfs_file_direct_write tail-call. |
| `swapfile_write_returns_etxtbsy` | INVARIANT | per-file_write: IS_SWAPFILE ⟹ -ETXTBSY. |
| `fsync_loop_terminates` | INVARIANT | per-file_fsync: redirtied_pages monotone ⟹ loop terminates. |
| `release_folio_no_rpc_under_reclaim` | INVARIANT | per-release_folio: kswapd/kcompactd ⟹ false (no RPC). |
| `read_io_serialized` | INVARIANT | per-file_read: nfs_start_io_read paired with nfs_end_io_read on every path. |
| `write_io_serialized` | INVARIANT | per-file_write: nfs_start_io_write paired with nfs_end_io_write on every path. |
| `mkwrite_returns_locked_or_sigbus` | INVARIANT | per-vm_page_mkwrite: success ⟹ VM_FAULT_LOCKED; incompatible-flush ⟹ VM_FAULT_SIGBUS. |
| `lock_reclaim_egrace` | INVARIANT | per-nfs_lock: FL_RECLAIM ⟹ -ENOGRACE. |
| `swap_activate_rejects_holes` | INVARIANT | per-swap_activate: blocks*512 < isize ⟹ -EINVAL. |

### Layer 2: TLA+

`fs/nfs/file.tla`:
- Per-open + per-read + per-write + per-fsync + per-commit-redirty + per-mkwrite.
- Properties:
  - `safety_check_flags_excludes_append_direct` — per-open: forbidden combo refused.
  - `safety_io_serializer_balanced` — per-read/write: start/end paired.
  - `safety_fsync_terminates` — per-fsync: bounded by redirtied_pages monotone.
  - `safety_release_folio_no_rpc_in_reclaim` — per-release_folio: reclaim context ⟹ false.
  - `safety_setlk_zaps_unless_delegation` — per-do_setlk: post-lock zap_caches unless delegation.
  - `liveness_per_write_eventually_committed` — per-write: dirty page ⟹ eventually nfs_commit_inode UNSTABLE→STABLE.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Nfs::check_flags` post: ret ∈ { 0, -EINVAL } | `Nfs::check_flags` |
| `NfsFile::open` post: success ⟹ FMODE_CAN_ODIRECT set | `NfsFile::open` |
| `NfsFile::read` post: ret > 0 ⟹ NFSIOS_NORMALREADBYTES bumped | `NfsFile::read` |
| `NfsFile::write` post: ret > 0 ⟹ NFSIOS_NORMALWRITTENBYTES bumped | `NfsFile::write` |
| `NfsFile::fsync` post: redirtied_pages stable | `NfsFile::fsync` |
| `NfsAops::write_begin` post: ret == 0 ⟹ folio locked + (uptodate ∨ once_thru-read-attempted) | `NfsAops::write_begin` |
| `NfsAops::write_end` post: nfs_update_folio called with offset, copied | `NfsAops::write_end` |
| `NfsAops::release_folio` post: private ∧ reclaim ⟹ false | `NfsAops::release_folio` |
| `NfsFileVm::page_mkwrite` post: ret ∈ { VM_FAULT_LOCKED, VM_FAULT_NOPAGE, VM_FAULT_SIGBUS, VM_FAULT_RETRY } | `NfsFileVm::page_mkwrite` |
| `NfsFile::lock` post: FL_RECLAIM ⟹ -ENOGRACE | `NfsFile::lock` |

### Layer 4: Verus/Creusot functional

`Per-VFS open → nfs_open_context allocate + close-to-open refresh → buffered I/O via page cache (read_iter / write_iter / mmap) → write staged into nfs_page → COMMIT RPC on flush/fsync/writeback → pNFS layoutcommit hook → cache coherency on locking (zap unless delegation)` semantic equivalence: per-Documentation/filesystems/nfs/ and RFC-1813 / RFC-7530.

### hardening

(Inherits row-1 features from `fs/nfs/00-overview.md` § Hardening.)

NFS-file reinforcement:

- **Per-O_APPEND|O_DIRECT rejected** — defense against per-undefined ordering under direct append.
- **Per-IS_SWAPFILE write blocked** — defense against per-corrupting active swap (-ETXTBSY).
- **Per-read/write IO serializer paired** — defense against per-direct-vs-buffered mix-tearing.
- **Per-revalidate-on-open (NFS_INO_INVALID_DATA)** — defense against per-stale-cache (close-to-open).
- **Per-revalidate-on-O_APPEND / beyond-EOF** — defense against per-extending-write-into-truncated-file.
- **Per-fsync redirty-stable loop** — defense against per-commit-races (re-dirty during COMMIT).
- **Per-pnfs_sync_inode in fsync** — defense against per-pNFS-layout-leak after fsync.
- **Per-FL_RECLAIM ENOGRACE** — defense against per-replay-lock-after-grace-period.
- **Per-do_setlk cache-zap unless delegation** — defense against per-stale-page-after-server-lock.
- **Per-do_unlk wb_all + iocounter_wait** — defense against per-unlock-vs-in-flight-write race.
- **Per-release_folio refuses RPC under reclaim** — defense against per-deadlock kswapd↔NFS-server.
- **Per-swap_activate holes check** — defense against per-corruption (sparse swap files).
- **Per-mkwrite folio_wait_writeback** — defense against per-COW-during-writeback torn page.
- **Per-mkwrite refuses cross-mapping folio** — defense against per-truncation-vs-mkwrite race.

