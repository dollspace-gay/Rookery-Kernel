---
title: "Tier-3: io_uring/rsrc.c — io_uring resource registration (fixed files + buffers)"
tags: ["tier-3", "io_uring", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`io_uring/rsrc.c` implements the **registered-resource** subsystem: per-fixed-file table, per-fixed-buffer table, per-tag, per-percpu-ref refcount, per-sparse-slot, per-skip-slot update, per-`IORING_REGISTER_FILES_UPDATE` / `_BUFFERS_UPDATE`, per-`IORING_REGISTER_CLONE_BUFFERS` (cross-ring share), per-`IORING_OP_FILES_UPDATE` SQE-driven slot rewrite, per-`io_buffer_register_bvec` (kernel-side bvec registration used by ublk/NVMe URING_CMD), per-huge-page-coalesce (`io_check_coalesce_buffer`/`io_coalesce_buffer`), per-`io_mapped_ubuf` (pinned-user-pages descriptor: ubuf, len, nr_bvecs, bvec[], folio_shift, dir, flags, release callback, refcount), per-`io_rsrc_node` (tag + refs + union {file_ptr, buf}), per-`io_rsrc_data` (table = `io_rsrc_node**` + nr). Per-mm-accounting via `__io_account_mem` / `__io_unaccount_mem` against `RLIMIT_MEMLOCK` and `mm.pinned_vm`. Per-`IORING_MAX_FIXED_FILES = 1<<20`, per-`IORING_MAX_REG_BUFFERS = 1<<14`. Critical for: per-syscall fd-table-lookup elimination (registered files), per-syscall iovec-copy elimination (registered buffers), per-MSG_ZEROCOPY-with-pinned-pages, per-cross-ring buffer sharing (KVM-vfio-userspace pattern), per-passthrough-block-IO via URING_CMD.

This Tier-3 covers `io_uring/rsrc.c` (~1563 lines).

### Acceptance Criteria

- [ ] AC-1: IORING_REGISTER_FILES with N fds: ctx.file_table.data.nr == N; each slot has an IoRsrcNode with file_ptr set; bitmap reflects occupied slots.
- [ ] AC-2: IORING_REGISTER_FILES exceeding IORING_MAX_FIXED_FILES (1<<20) returns -EMFILE.
- [ ] AC-3: IORING_REGISTER_FILES with fd == -1 produces a sparse slot (NULL node) and tag must be 0 (else -EINVAL).
- [ ] AC-4: IORING_REGISTER_FILES with an io_uring fd returns -EBADF (no recursion).
- [ ] AC-5: IORING_REGISTER_FILES_UPDATE with IORING_REGISTER_FILES_SKIP leaves the slot untouched.
- [ ] AC-6: IORING_REGISTER_BUFFERS pins pages, charges RLIMIT_MEMLOCK + mm.pinned_vm; unregister reverses both.
- [ ] AC-7: Huge-page-backed buffer triggers io_check_coalesce_buffer → io_coalesce_buffer; nr_bvecs drops to folio count; folio_shift updated.
- [ ] AC-8: validate_fixed_range rejects buf_addr below imu.ubuf or beyond ubuf+len with -EFAULT.
- [ ] AC-9: io_import_fixed with wrong direction (e.g. read against IO_IMU_SOURCE-only buf) returns -EFAULT.
- [ ] AC-10: IORING_OP_FILES_UPDATE with offset = IORING_FILE_INDEX_ALLOC dynamically installs into next free slot; resulting slot index returned via put_user.
- [ ] AC-11: IORING_REGISTER_CLONE_BUFFERS copies src ring's buf_table; imu.refs incremented (shared); accounting not double-charged.
- [ ] AC-12: IORING_REGISTER_CLONE_BUFFERS with DST_REPLACE on a non-empty dst frees old table and installs cloned table.
- [ ] AC-13: io_free_rsrc_node with node.tag != 0 posts an aux CQE with res=0, user_data=tag.
- [ ] AC-14: io_buffer_register_bvec installs a kernel-side bvec (IO_REGBUF_F_KBUF); io_buffer_unregister_bvec only unregisters IO_REGBUF_F_KBUF slots (-EBUSY for user-registered).

### Architecture

```
struct IoRsrcNode {
  type: u32,                                  // IORING_RSRC_FILE | _BUFFER
  refs: i32,                                  // open-count
  tag: u64,                                   // userspace opaque (CQE on release)
  payload: union {
    file_ptr: usize,                          // file* | flags (low bits)
    buf: *IoMappedUbuf,
  },
}

struct IoRsrcData {
  nodes: *mut *mut IoRsrcNode,                // sparse table
  nr: u32,
}

struct IoMappedUbuf {
  refs: AtomicU32,                            // refcount_t for clone-sharing
  ubuf: u64,                                  // user va base (0 for KBUF)
  len: u64,
  nr_bvecs: u32,
  acct_pages: u32,                            // RLIMIT_MEMLOCK charge
  folio_shift: u8,                            // log2(folio_size)
  flags: u8,                                  // IO_REGBUF_F_KBUF
  dir: u8,                                    // IO_IMU_DEST | IO_IMU_SOURCE
  release: fn(*mut ()),
  priv: *mut (),                              // self (user) | rq (kbuf)
  bvec: [BioVec; _],                          // flexible array
}

struct IoRsrcUpdate {
  file: *File,
  arg: u64,                                   // user fds[] or iovec[] ptr
  nr_args: u32,
  offset: u32,                                // base index or IORING_FILE_INDEX_ALLOC
}

const IORING_MAX_FIXED_FILES: u32 = 1 << 20;
const IORING_MAX_REG_BUFFERS: u32 = 1 << 14;
const IO_CACHED_BVECS_SEGS:   u32 = 32;
```

`Rsrc::sqe_files_register(ctx, fds, nr, tags) -> Result<()>`:
1. require !ctx.file_table.data.nr (else -EBUSY).
2. enforce nr bounds (1..=IORING_MAX_FIXED_FILES, ≤ RLIMIT_NOFILE).
3. alloc table.
4. for each slot:
   - copy fd/tag.
   - sparse (fd == -1): require !tag; continue.
   - fget; reject !file or io_is_uring_fops.
   - alloc node; set tag if any; install via io_fixed_file_set.
5. on fail: clear_table_tags + unregister.

`Rsrc::sqe_files_update(ctx, up, nr_args) -> i32`:
1. require ctx.file_table.data.nr.
2. for each slot:
   - copy fd/tag.
   - skip-sentinel (IORING_REGISTER_FILES_SKIP): continue.
   - i = array_index_nospec(up.offset + done).
   - reset old (put old node + clear bitmap).
   - if fd != -1: fget + alloc node + install.
3. return done > 0 ? done : err (partial-progress preserved).

`Rsrc::sqe_buffer_register(ctx, iov, last_hpage) -> Result<*IoRsrcNode>`:
1. sparse: !iov.iov_base ∧ !iov.iov_len → NULL.
2. validate_user_buf_range (≤ 1GiB, !overflow).
3. alloc node + io_pin_pages.
4. coalesce_buffer if huge-page eligible.
5. alloc imu; account_pin (RLIMIT_MEMLOCK + pinned_vm).
6. populate imu.{ubuf, len, folio_shift, flags, dir, release, priv, refs=1}.
7. fill bvec[] using folio_shift.
8. node.buf = imu.

`Rsrc::sqe_buffers_register(ctx, uvec, nr_args, tags) -> Result<()>`:
1. -EBUSY if ctx.buf_table.nr.
2. bounds check (1..=IORING_MAX_REG_BUFFERS).
3. data_alloc.
4. iterate uvec, calling sqe_buffer_register; sparse + tag → -EINVAL.
5. on fail: clear_table_tags + unregister.

`Rsrc::register_clone_buffers(ctx, arg) -> Result<()>`:
1. copy buf; reject unknown flags; require zero pad.
2. -EBUSY unless DST_REPLACE or empty dst.
3. resolve src_fd via io_uring_ctx_get_file (registered or unregistered).
4. if src != ctx:
   - drop ctx.uring_lock; lock_two_rings in pointer order.
   - require !src.submitter_task or src.submitter_task == current.
5. clone_buffers:
   - require same user + mm_account.
   - alloc temp table sized max(new, old).
   - copy unchanged prefix/suffix (refs++).
   - for each cloned slot: alloc node, refcount_inc on src imu, share imu (no re-pin, no re-account).
   - if DST_REPLACE: free old table (decs old imu refs).
   - install new table.

`Rsrc::find_buf_node(req, issue_flags) -> Option<*IoRsrcNode>`:
1. cached (REQ_F_BUF_NODE)?: return cache.
2. submit_lock.
3. lookup ctx.buf_table[req.buf_index].
4. if hit: refs++; cache on req; return.

`Rsrc::import_reg_buf(req, iter, buf_addr, len, ddir, issue_flags) -> Result<()>`:
1. find_buf_node ?: -EFAULT.
2. import_fixed(ddir, iter, node.buf, buf_addr, len):
   - validate_fixed_range.
   - direction check.
   - KBUF path: io_import_kbuf.
   - user path: walk bvec using folio_shift fast path.

`Rsrc::buffer_register_bvec(cmd, rq, release, index, issue_flags) -> Result<()>` (EXPORT_SYMBOL_GPL):
1. submit_lock.
2. bounds + array_index_nospec.
3. -EBUSY if occupied.
4. alloc node + imu sized for blk_rq_nr_phys_segments.
5. populate imu (ubuf=0, len=blk_rq_bytes, flags=IO_REGBUF_F_KBUF, dir = 1 << rq_data_dir, priv=rq).
6. fill bvec from rq_for_each_bvec.
7. install in slot.

`Rsrc::files_update(req, issue_flags) -> IoComplete`:
1. if up.offset == IORING_FILE_INDEX_ALLOC: files_update_with_index_alloc (dynamic).
2. else: submit_lock; register_rsrc_update(FILE, &up2, nr_args); unlock.
3. req_set_fail on error; set_res; IOU_COMPLETE.

### Out of Scope

- io_uring/filetable.c fixed-file bitmap/range/install primitives (covered separately in `filetable.md` Tier-3).
- io_uring/memmap.c io_uring mmap regions (covered in `memmap.md` Tier-3).
- io_uring/register.c top-level registration multiplexer (covered in `register.md` Tier-3).
- io_uring/kbuf.c provided-buffer rings (covered in `kbuf.md` Tier-3).
- io_uring/openclose.c fixed-fd install/close (covered in `openclose.md` Tier-3).
- mm/gup.c io_pin_pages / unpin_user_folio primitives (covered in `mm/` Tier-3 docs).
- net/socket.c MSG_ZEROCOPY skb_ubuf integration (covered in `net/` Tier-3 docs).
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_rsrc_node` | per-slot descriptor (tag + refs + file_ptr/buf) | `IoRsrcNode` |
| `struct io_rsrc_data` | per-table { nodes: *IoRsrcNode[], nr } | `IoRsrcData` |
| `struct io_mapped_ubuf` | per-pinned-buffer descriptor | `IoMappedUbuf` |
| `struct io_rsrc_update` | per-IORING_OP_FILES_UPDATE state | `IoRsrcUpdate` |
| `io_rsrc_node_alloc()` | per-slot alloc from `ctx.node_cache` | `Rsrc::node_alloc` |
| `io_rsrc_cache_init()` / `_free()` | per-ctx node + imu slab init/destroy | `Rsrc::cache_init` / `cache_free` |
| `io_free_rsrc_node()` | per-slot release (fput file or buffer_unmap) | `Rsrc::free_rsrc_node` |
| `io_put_rsrc_node()` (header) | per-refcount dec → free | `Rsrc::put_node` |
| `io_rsrc_node_lookup()` (header) | per-index slot lookup | `Rsrc::lookup` |
| `io_reset_rsrc_node()` (header) | per-slot replace (put old, NULL) | `Rsrc::reset_node` |
| `io_rsrc_data_alloc()` / `_free()` | per-table backing alloc | `Rsrc::data_alloc` / `data_free` |
| `io_clear_table_tags()` | per-fail rollback: clear tags | `Rsrc::clear_table_tags` |
| `io_account_mem()` / `io_unaccount_mem()` | per-RLIMIT_MEMLOCK + pinned_vm | `Rsrc::account_mem` / `unaccount_mem` |
| `__io_account_mem()` / `__io_unaccount_mem()` | per-user.locked_vm cmpxchg | `Rsrc::user_account` / `user_unaccount` |
| `io_validate_user_buf_range()` | per-user-range sanity (≤ 1GiB, !overflow) | `Rsrc::validate_user_buf_range` |
| `io_alloc_imu()` / `io_free_imu()` | per-imu slab vs kvmalloc dispatch | `Rsrc::alloc_imu` / `free_imu` |
| `io_release_ubuf()` | per-imu cleanup: unpin folios | `Rsrc::release_ubuf` |
| `io_buffer_unmap()` | per-imu dec-refs + unaccount + release | `Rsrc::buffer_unmap` |
| `io_sqe_buffer_register()` | per-iovec → io_rsrc_node | `Rsrc::sqe_buffer_register` |
| `io_sqe_buffers_register()` | per-IORING_REGISTER_BUFFERS handler | `Rsrc::sqe_buffers_register` |
| `io_sqe_buffers_unregister()` | per-IORING_UNREGISTER_BUFFERS | `Rsrc::sqe_buffers_unregister` |
| `io_sqe_files_register()` | per-IORING_REGISTER_FILES handler | `Rsrc::sqe_files_register` |
| `io_sqe_files_unregister()` | per-IORING_UNREGISTER_FILES | `Rsrc::sqe_files_unregister` |
| `__io_sqe_files_update()` | per-IORING_REGISTER_FILES_UPDATE inner | `Rsrc::sqe_files_update` |
| `__io_sqe_buffers_update()` | per-IORING_REGISTER_BUFFERS_UPDATE inner | `Rsrc::sqe_buffers_update` |
| `__io_register_rsrc_update()` | per-update dispatch by type | `Rsrc::register_rsrc_update` |
| `io_register_files_update()` | per-old-API IORING_REGISTER_FILES_UPDATE | `Rsrc::register_files_update` |
| `io_register_rsrc_update()` | per-new-API _UPDATE2 | `Rsrc::register_rsrc_update_v2` |
| `io_register_rsrc()` | per-IORING_REGISTER_BUFFERS2 / _FILES2 | `Rsrc::register_rsrc` |
| `io_files_update_prep()` / `io_files_update()` | per-IORING_OP_FILES_UPDATE SQE | `Rsrc::files_update_prep` / `files_update` |
| `io_files_update_with_index_alloc()` | per-IORING_FILE_INDEX_ALLOC dynamic | `Rsrc::files_update_with_index_alloc` |
| `io_register_clone_buffers()` | per-IORING_REGISTER_CLONE_BUFFERS | `Rsrc::register_clone_buffers` |
| `io_clone_buffers()` | per-cross-ring buf-table clone inner | `Rsrc::clone_buffers` |
| `lock_two_rings()` | per-two-ctx mutex order | `Rsrc::lock_two_rings` |
| `headpage_already_acct()` | per-hugepage-dedupe accounting | `Rsrc::headpage_already_acct` |
| `io_buffer_account_pin()` | per-buf RLIMIT/pinned_vm accounting | `Rsrc::buffer_account_pin` |
| `io_check_coalesce_buffer()` / `io_coalesce_buffer()` | per-huge-page bvec coalesce | `Rsrc::check_coalesce_buffer` / `coalesce_buffer` |
| `io_buffer_register_bvec()` | per-kernel-side bvec slot install (ublk/NVMe) | `Rsrc::buffer_register_bvec` |
| `io_buffer_unregister_bvec()` | per-kernel-side bvec slot remove | `Rsrc::buffer_unregister_bvec` |
| `validate_fixed_range()` | per-fixed-buf range vs imu bounds | `Rsrc::validate_fixed_range` |
| `io_import_kbuf()` | per-IO_REGBUF_F_KBUF bvec-iter import | `Rsrc::import_kbuf` |
| `io_import_fixed()` | per-user fixed-buf bvec-iter import | `Rsrc::import_fixed` |
| `io_find_buf_node()` | per-issue: lookup + cache buf_node on req | `Rsrc::find_buf_node` |
| `io_import_reg_buf()` | per-issue: import single fixed buf | `Rsrc::import_reg_buf` |
| `io_vec_free()` / `io_vec_realloc()` | per-iou_vec slab management | `Rsrc::vec_free` / `vec_realloc` |
| `io_vec_fill_bvec()` / `io_vec_fill_kern_bvec()` | per-registered-iovec → bio_vec | `Rsrc::vec_fill_bvec` / `vec_fill_kern_bvec` |
| `io_estimate_bvec_size()` / `io_kern_bvec_size()` / `iov_kern_bvec_size()` | per-bvec sizing | `Rsrc::estimate_bvec_size` / `kern_bvec_size` |
| `io_import_reg_vec()` | per-issue: import vectored fixed buf | `Rsrc::import_reg_vec` |
| `io_prep_reg_iovec()` | per-prep: validate uvec + realloc | `Rsrc::prep_reg_iovec` |
| `IORING_MAX_FIXED_FILES` (=1<<20) | per-fixed-file cap | const |
| `IORING_MAX_REG_BUFFERS` (=1<<14) | per-buffer cap | const |
| `IO_CACHED_BVECS_SEGS` (=32) | per-imu inline-bvec count | const |
| `IORING_RSRC_FILE` / `_BUFFER` | per-rsrc type tag | shared |
| `IORING_RSRC_REGISTER_SPARSE` | per-sparse register flag | shared |
| `IORING_REGISTER_FILES_SKIP` | per-update: skip slot sentinel | shared |
| `IORING_REGISTER_SRC_REGISTERED` | per-clone src is registered fd | shared |
| `IORING_REGISTER_DST_REPLACE` | per-clone replace dst table | shared |
| `IORING_FILE_INDEX_ALLOC` | per-OP_FILES_UPDATE alloc-mode sentinel | shared |
| `IO_REGBUF_F_KBUF` | per-kernel-side bvec-registered flag | shared |
| `IO_IMU_DEST` / `IO_IMU_SOURCE` | per-imu direction bits | shared |

### compatibility contract

REQ-1: struct io_rsrc_node (per-slot):
- type: u32 — IORING_RSRC_FILE | _BUFFER.
- refs: int — open-count (incremented by io_find_buf_node etc., decremented by put).
- tag: u64 — userspace-supplied opaque tag; emitted on async release via io_post_aux_cqe.
- file_ptr (FILE) / buf (BUFFER) — union payload.

REQ-2: struct io_rsrc_data (per-table):
- nodes: `IoRsrcNode**` — sparse array (NULL = empty slot).
- nr: u32 — table length.

REQ-3: struct io_mapped_ubuf (per-buffer):
- ubuf: u64 — original user va base (or 0 for kernel bvec).
- len: u64 — user-supplied length.
- nr_bvecs: int — bvec[] count.
- acct_pages: int — pages charged against RLIMIT_MEMLOCK.
- folio_shift: u8 — log2(folio_size); PAGE_SHIFT or larger for huge.
- flags: u8 — IO_REGBUF_F_KBUF (kernel-side bvec, e.g. block request).
- dir: u8 — IO_IMU_DEST | IO_IMU_SOURCE allowed-direction mask.
- refs: refcount_t — for cross-ring clone (refcount_inc/dec/test).
- release: fn(priv) — cleanup (io_release_ubuf for user pages; per-cmd for kernel).
- priv: *void — release context.
- bvec: bio_vec[] — flexible array.

REQ-4: struct io_rsrc_update (per-SQE op):
- file: *File (carrier).
- arg: u64 — userspace pointer to fds[] (or iovec[]).
- nr_args: u32 — slot count.
- offset: u32 — base slot index (or IORING_FILE_INDEX_ALLOC).

REQ-5: io_account_mem / io_unaccount_mem:
- account_mem(user, mm_account, nr_pages):
  - if user: __io_account_mem(user, nr_pages) — cmpxchg-loop on user.locked_vm vs (RLIMIT_MEMLOCK >> PAGE_SHIFT); -ENOMEM if exceeded.
  - if mm_account: atomic64_add(nr_pages, &mm_account.pinned_vm).
- unaccount_mem(user, mm_account, nr_pages):
  - if user: __io_unaccount_mem(user, nr_pages) — decrement.
  - if mm_account: atomic64_sub(nr_pages, &mm_account.pinned_vm).

REQ-6: io_validate_user_buf_range(uaddr, ulen):
- ulen > SZ_1G ∨ !ulen → -EFAULT.
- base + PAGE_ALIGN(ulen) overflow check → -EOVERFLOW.

REQ-7: io_rsrc_node_alloc(ctx, type):
- node = io_cache_alloc(&ctx.node_cache, GFP_KERNEL).
- if node: node.type = type; node.refs = 1; node.tag = 0; node.file_ptr = 0.

REQ-8: io_rsrc_cache_init / io_rsrc_cache_free:
- imu_cache: io_alloc_cache_init(struct_size(io_mapped_ubuf, bvec, IO_CACHED_BVECS_SEGS)).
- node_cache: io_alloc_cache_init(sizeof(io_rsrc_node)).
- both with IO_ALLOC_CACHE_MAX entries.

REQ-9: io_rsrc_data_alloc(data, nr):
- data.nodes = kvmalloc_objs(IoRsrcNode*, nr, GFP_KERNEL_ACCOUNT | __GFP_ZERO).
- on fail: -ENOMEM.
- else: data.nr = nr.

REQ-10: io_rsrc_data_free(ctx, data):
- if !data.nr: return.
- while data.nr--: if data.nodes[i]: io_put_rsrc_node(ctx, ...).
- kvfree(data.nodes); data.nodes = NULL; data.nr = 0.

REQ-11: io_free_rsrc_node(ctx, node):
- if node.tag: io_post_aux_cqe(ctx, node.tag, 0, 0) — emit tag CQE.
- type switch:
  - IORING_RSRC_FILE: fput(io_slot_file(node)).
  - IORING_RSRC_BUFFER: io_buffer_unmap(ctx, node.buf).
  - default: WARN_ON_ONCE.
- io_cache_free(&ctx.node_cache, node).

REQ-12: io_buffer_unmap(ctx, imu):
- if refcount > 1: dec; if still > 0: return.
- if imu.acct_pages: io_unaccount_mem(ctx.user, ctx.mm_account, acct_pages).
- imu.release(imu.priv).
- io_free_imu(ctx, imu).

REQ-13: io_sqe_files_register(ctx, fds[], nr_args, tags[]):
- if ctx.file_table.data.nr: -EBUSY.
- if !nr_args: -EINVAL.
- if nr_args > IORING_MAX_FIXED_FILES ∨ > RLIMIT_NOFILE: -EMFILE.
- io_alloc_file_tables(ctx, &ctx.file_table, nr_args).
- for i in 0..nr_args:
  - copy_from_user tag[i], fd[i].
  - sparse (fd == -1 ∨ !fds): if tag: -EINVAL; continue.
  - file = fget(fd); if !file: -EBADF.
  - if io_is_uring_fops(file): fput; -EBADF (reject recursive uring).
  - node = io_rsrc_node_alloc(ctx, IORING_RSRC_FILE); if !node: fput; -ENOMEM.
  - if tag: node.tag = tag.
  - ctx.file_table.data.nodes[i] = node; io_fixed_file_set(node, file); io_file_bitmap_set.
- io_file_table_set_alloc_range(ctx, 0, nr); return 0.
- on fail: io_clear_table_tags + io_sqe_files_unregister.

REQ-14: __io_sqe_files_update(ctx, up, nr_args):
- if !ctx.file_table.data.nr: -ENXIO.
- if up.offset + nr_args > data.nr: -EINVAL.
- for done in 0..nr_args:
  - copy_from_user tag, fd.
  - if (fd == IORING_REGISTER_FILES_SKIP ∨ fd == -1) ∧ tag: -EINVAL.
  - if fd == IORING_REGISTER_FILES_SKIP: continue (preserve slot).
  - i = array_index_nospec(up.offset + done, data.nr).
  - if io_reset_rsrc_node: io_file_bitmap_clear.
  - if fd != -1:
    - file = fget(fd); reject !file or io_is_uring_fops.
    - node = io_rsrc_node_alloc(IORING_RSRC_FILE); on fail: fput; -ENOMEM.
    - data.nodes[i] = node; if tag: node.tag = tag; io_fixed_file_set; io_file_bitmap_set.
- return done > 0 ? done : err.

REQ-15: io_sqe_buffers_register(ctx, uvec[], nr_args, tags[]):
- BUILD_BUG_ON(IORING_MAX_REG_BUFFERS >= 1u<<16) (must fit u16 buf_index).
- if ctx.buf_table.nr: -EBUSY.
- if !nr_args ∨ > IORING_MAX_REG_BUFFERS: -EINVAL.
- io_rsrc_data_alloc(&data, nr_args).
- for i in 0..nr_args:
  - if arg: iovec_from_user(uvec, 1, 1, &fast_iov, compat); iov-step (sizeof iovec or compat_iovec).
  - else: zeroed iov (sparse).
  - if tags: copy_from_user tag.
  - node = io_sqe_buffer_register(ctx, iov, &last_hpage); if IS_ERR: break.
  - if tag: require node != NULL (sparse + tag invalid) → -EINVAL.
  - data.nodes[i] = node.
- ctx.buf_table = data.
- on fail: io_clear_table_tags + io_sqe_buffers_unregister.

REQ-16: io_sqe_buffer_register(ctx, iov, &last_hpage) → *IoRsrcNode | ERR_PTR:
- if !iov.iov_base: if iov.iov_len: -EFAULT; else: return NULL (sparse).
- io_validate_user_buf_range(iov_base, iov_len).
- node = io_rsrc_node_alloc(IORING_RSRC_BUFFER).
- pages = io_pin_pages(iov.iov_base, iov.iov_len, &nr_pages) — pins user pages.
- if nr_pages > 1 ∧ io_check_coalesce_buffer: if nr_pages_mid != 1: io_coalesce_buffer (collapse to head-pages only).
- imu = io_alloc_imu(ctx, nr_pages); inline for ≤ IO_CACHED_BVECS_SEGS else kvmalloc.
- imu.nr_bvecs = nr_pages.
- io_buffer_account_pin(ctx, pages, nr_pages, imu, &last_hpage) — RLIMIT_MEMLOCK + pinned_vm.
- imu.ubuf = iov_base; imu.len = iov_len; imu.folio_shift = PAGE_SHIFT (or data.folio_shift if coalesced).
- imu.release = io_release_ubuf; imu.priv = imu; imu.flags = 0.
- imu.dir = IO_IMU_DEST | IO_IMU_SOURCE.
- refcount_set(&imu.refs, 1).
- off = iov_base & ~PAGE_MASK (+ data.first_folio_page_idx<<PAGE_SHIFT if coalesced).
- node.buf = imu.
- fill bvec[i] = (pages[i], min(size, folio_size - off), off); off=0 after first.
- on fail: io_free_imu; unpin_user_folio loop; io_cache_free(node_cache).

REQ-17: __io_sqe_buffers_update(ctx, up, nr_args):
- require ctx.buf_table.nr; require offset+nr_args ≤ nr.
- iter:
  - iov = iovec_from_user(uvec, 1, 1, &fast_iov, compat).
  - copy tag.
  - node = io_sqe_buffer_register(ctx, iov, &last_hpage).
  - if tag ∧ !node: -EINVAL.
  - i = array_index_nospec(offset+done, nr); io_reset_rsrc_node(buf_table, i); buf_table.nodes[i] = node.
  - user_data += compat ? sizeof(compat_iovec) : sizeof(iovec).

REQ-18: __io_register_rsrc_update(ctx, type, up, nr_args):
- lockdep_assert_held(ctx.uring_lock).
- check_add_overflow(up.offset, nr_args).
- type switch: FILE → __io_sqe_files_update; BUFFER → __io_sqe_buffers_update; else -EINVAL.

REQ-19: io_register_files_update(ctx, arg, nr_args) (old API):
- !nr_args: -EINVAL.
- copy_from_user(up, arg, sizeof(io_uring_rsrc_update)) (smaller struct).
- if up.resv ∨ up.resv2: -EINVAL.
- delegate via __io_register_rsrc_update(IORING_RSRC_FILE, ...).

REQ-20: io_register_rsrc_update(ctx, arg, size, type) (new API _UPDATE2):
- require size == sizeof(io_uring_rsrc_update2).
- copy_from_user up.
- require up.nr ∧ !up.resv ∧ !up.resv2.

REQ-21: io_register_rsrc(ctx, arg, size, type) (REGISTER_BUFFERS2/_FILES2):
- require size == sizeof(io_uring_rsrc_register).
- reject rr.resv2; require rr.nr.
- allowed flags: IORING_RSRC_REGISTER_SPARSE.
- if FILE: sparse-only (no data); else: io_sqe_files_register(rr.data, rr.nr, rr.tags).
- if BUFFER: same pattern → io_sqe_buffers_register.

REQ-22: io_files_update_prep / io_files_update (IORING_OP_FILES_UPDATE):
- prep: reject REQ_F_FIXED_FILE / REQ_F_BUFFER_SELECT; reject rw_flags / splice_fd_in.
- up.offset = sqe.off; up.nr_args = sqe.len (require > 0); up.arg = sqe.addr.
- issue:
  - if up.offset == IORING_FILE_INDEX_ALLOC: io_files_update_with_index_alloc (dynamic install, put_user the resulting fixed-slot index).
  - else: io_ring_submit_lock; __io_register_rsrc_update(FILE, &up2, up.nr_args); io_ring_submit_unlock.
- if ret < 0: req_set_fail; io_req_set_res; IOU_COMPLETE.

REQ-23: io_files_update_with_index_alloc(req, issue_flags):
- require ctx.file_table.data.nr (else -ENXIO).
- for done in 0..up.nr_args:
  - get_user fd; fget(fd); io_fixed_fd_install(req, issue_flags, file, IORING_FILE_INDEX_ALLOC).
  - if ok: put_user(slot_index, &fds[done]); on put-fault: __io_close_fixed; -EFAULT.
- return done > 0 ? done : ret.

REQ-24: io_buffer_register_bvec(cmd, rq, release, index, issue_flags) (EXPORTED):
- ctx = cmd_to_io_kiocb(cmd).ctx; data = &ctx.buf_table.
- io_ring_submit_lock.
- require index < data.nr; index = array_index_nospec.
- require !data.nodes[index] (else -EBUSY).
- node = io_rsrc_node_alloc(IORING_RSRC_BUFFER).
- imu = io_alloc_imu(ctx, blk_rq_nr_phys_segments(rq)).
- imu.ubuf = 0; imu.len = blk_rq_bytes(rq); imu.acct_pages = 0; imu.folio_shift = PAGE_SHIFT.
- imu.release = release; imu.priv = rq; imu.flags = IO_REGBUF_F_KBUF.
- imu.dir = 1 << rq_data_dir(rq) (READ → IO_IMU_DEST or WRITE → IO_IMU_SOURCE).
- rq_for_each_bvec: imu.bvec[nr_bvecs++] = bv.
- imu.nr_bvecs = nr_bvecs.
- node.buf = imu; data.nodes[index] = node.
- unlock; return 0.

REQ-25: io_buffer_unregister_bvec(cmd, index, issue_flags) (EXPORTED):
- require index < data.nr; node = data.nodes[index].
- require IO_REGBUF_F_KBUF (else -EBUSY — won't yank a user-registered buf).
- io_put_rsrc_node(ctx, node); data.nodes[index] = NULL.

REQ-26: validate_fixed_range(buf_addr, len, imu):
- buf_end overflow check.
- buf_addr ≥ imu.ubuf ∧ buf_end ≤ imu.ubuf + imu.len.
- len ≤ MAX_RW_COUNT.

REQ-27: io_import_fixed(ddir, iter, imu, buf_addr, len):
- validate_fixed_range.
- require imu.dir & (1 << ddir) (else -EFAULT — wrong-direction reject).
- if !len: iov_iter_bvec(NULL, 0); return 0.
- offset = buf_addr - imu.ubuf.
- if IO_REGBUF_F_KBUF: io_import_kbuf (iov_iter_bvec with offset advance).
- else (user-pinned): walk imu.bvec using folio-shift fast path (skip whole-folios then advance into target).
- iov_iter_bvec(iter, ddir, bvec, nr_segs, len); iter.iov_offset = offset.

REQ-28: io_find_buf_node(req, issue_flags):
- if REQ_F_BUF_NODE: return cached req.buf_node.
- req.flags |= REQ_F_BUF_NODE.
- io_ring_submit_lock.
- node = io_rsrc_node_lookup(&ctx.buf_table, req.buf_index).
- if node: node.refs++; req.buf_node = node; unlock; return.
- else: clear REQ_F_BUF_NODE; unlock; return NULL.

REQ-29: io_import_reg_buf(req, iter, buf_addr, len, ddir, issue_flags):
- node = io_find_buf_node(req, issue_flags) ?: -EFAULT.
- io_import_fixed(ddir, iter, node.buf, buf_addr, len).

REQ-30: io_clone_buffers(ctx, src_ctx, arg):
- lockdep_assert_held both ctx + src_ctx uring_lock.
- require same user + mm_account (-EINVAL else).
- require nr ∨ !(dst_off ∨ src_off).
- require !ctx.buf_table.nr ∨ DST_REPLACE.
- nbufs = src.buf_table.nr; arg.nr default = nbufs; cap ≤ IORING_MAX_REG_BUFFERS.
- overflow-check src_off+nr, dst_off+nr.
- io_rsrc_data_alloc(&data, max(nbufs, ctx.buf_table.nr)).
- copy original dst nodes [0..dst_off): node.refs++; data.nodes[i] = node.
- for nr buffers: src_node = lookup(src, i); if !src_node: dst_node = NULL; else: alloc new IoRsrcNode + refcount_inc(&src_node.buf.refs); dst_node.buf = src_node.buf. data.nodes[off++] = dst_node; i++.
- copy original dst nodes [dst_off+nr..ctx.buf_table.nr): node.refs++; data.nodes[i] = node.
- if DST_REPLACE: io_rsrc_data_free(ctx, &ctx.buf_table) (decs old refs).
- WARN_ON_ONCE(ctx.buf_table.nr); ctx.buf_table = data.

REQ-31: io_register_clone_buffers(ctx, arg) (IORING_REGISTER_CLONE_BUFFERS):
- copy_from_user buf.
- allowed flags: SRC_REGISTERED | DST_REPLACE.
- if !DST_REPLACE ∧ ctx.buf_table.nr: -EBUSY.
- require !memchr_inv(buf.pad, 0) (zero pad).
- registered_src = (flags & SRC_REGISTERED) != 0.
- file = io_uring_ctx_get_file(buf.src_fd, registered_src); IS_ERR → propagate.
- src_ctx = file.private_data.
- if src_ctx != ctx:
  - mutex_unlock(ctx.uring_lock); lock_two_rings(ctx, src_ctx).
  - if src.submitter_task ∧ src.submitter_task != current: -EEXIST.
- io_clone_buffers(ctx, src_ctx, &buf).
- if src_ctx != ctx: mutex_unlock(src.uring_lock).
- if !registered_src: fput(file).

REQ-32: lock_two_rings(ctx1, ctx2):
- enforce mutex order via pointer compare (swap if ctx1 > ctx2); lock ctx1, then ctx2 with SINGLE_DEPTH_NESTING.

REQ-33: io_vec_free / io_vec_realloc (iou_vec):
- vec_free: kfree(iv.iovec); iv.iovec = NULL; iv.nr = 0.
- vec_realloc: kmalloc_objs(iov[0], nr_entries, GFP_KERNEL_ACCOUNT|__GFP_NOWARN); free old; assign.

REQ-34: io_vec_fill_bvec(ddir, iter, imu, iovec, nr_iovs, vec):
- iterate iovs; validate_fixed_range each; track total_len.
- offset = (iov.base - imu.ubuf) + imu.bvec[0].bv_offset.
- src_bvec = imu.bvec + (offset >> folio_shift); offset &= folio_mask.
- fill res_bvec from src_bvec with seg_size = min(iov_len, folio_size - offset).
- total_len ≤ MAX_RW_COUNT (else -EINVAL).
- iov_iter_bvec(iter, ddir, res_bvec, bvec_idx, total_len).

REQ-35: io_vec_fill_kern_bvec / io_kern_bvec_size / iov_kern_bvec_size (IO_REGBUF_F_KBUF path):
- treat iov.iov_base as byte-offset into imu.bvec.
- bvec_iter_advance + for_each_mp_bvec → fill res_bvec.

REQ-36: io_import_reg_vec(ddir, iter, req, vec, nr_iovs, issue_flags):
- find_buf_node.
- check direction.
- iovec_off = vec.nr - nr_iovs; iov = vec.iovec + iovec_off.
- nr_segs = (kbuf ? io_kern_bvec_size : io_estimate_bvec_size).
- if sizeof(bio_vec) > sizeof(iovec): scale nr_segs to fit (bvec_bytes / iovec_size + nr_iovs).
- if nr_segs > vec.nr: realloc tmp_vec; memcpy iov; io_vec_free(vec); *vec = tmp_vec; REQ_F_NEED_CLEANUP.
- dispatch: kbuf → vec_fill_kern_bvec; else vec_fill_bvec.

REQ-37: io_prep_reg_iovec(req, iv, uvec, uvec_segs):
- if uvec_segs > iv.nr: io_vec_realloc; REQ_F_NEED_CLEANUP.
- iovec_off = iv.nr - uvec_segs; iov = iv.iovec + iovec_off.
- iovec_from_user(uvec, uvec_segs, uvec_segs, iov, compat).
- req.flags |= REQ_F_IMPORT_BUFFER.

REQ-38: Huge-page coalesce — io_check_coalesce_buffer:
- folio = page_folio(page_array[0]); nr_pages_mid = folio_nr_pages(folio); folio_shift = folio_shift(folio); first_folio_page_idx = folio_page_idx(folio, page_array[0]).
- walk pages; require contiguous-within-folio and all folios same size (head/tail may differ in page count).
- on success: data.{nr_folios, nr_pages_head, nr_pages_mid, folio_shift} populated.

REQ-39: io_coalesce_buffer (after check_coalesce_buffer):
- allocate new_array = kvmalloc_objs(struct page*, nr_folios) — head-page-only.
- for each folio: take head page p; drop nr-1 refs (`unpin_user_folio(folio, nr-1)`); new_array[i] = p.
- kvfree old page_array; *pages = new_array; *nr_pages = nr_folios.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `register_files_bounds` | INVARIANT | per-sqe_files_register: nr_args ∈ (0, IORING_MAX_FIXED_FILES] ∧ ≤ RLIMIT_NOFILE. |
| `register_buffers_bounds` | INVARIANT | per-sqe_buffers_register: nr_args ∈ (0, IORING_MAX_REG_BUFFERS]. |
| `no_uring_in_file_table` | INVARIANT | per-register/update: io_is_uring_fops ⟹ -EBADF. |
| `validate_fixed_range_bounds` | INVARIANT | per-import: buf_addr ≥ imu.ubuf ∧ buf_end ≤ imu.ubuf+imu.len ∧ len ≤ MAX_RW_COUNT. |
| `fixed_buf_dir_check` | INVARIANT | per-import_fixed: imu.dir & (1 << ddir). |
| `mem_account_balanced` | INVARIANT | per-register/unregister: net delta on user.locked_vm + mm.pinned_vm = 0. |
| `imu_refcount_balanced` | INVARIANT | per-clone+free: refcount_inc / dec_and_test pairs preserve invariant. |
| `node_tag_aux_cqe_emitted` | INVARIANT | per-free_rsrc_node: node.tag ≠ 0 ⟹ io_post_aux_cqe called with that tag. |
| `clone_same_user_and_mm` | INVARIANT | per-clone_buffers: ctx.user == src.user ∧ ctx.mm_account == src.mm_account. |
| `dst_replace_or_empty` | INVARIANT | per-clone_buffers: ctx.buf_table.nr == 0 ∨ DST_REPLACE set. |
| `skip_sentinel_no_tag` | INVARIANT | per-files_update: (fd == FILES_SKIP ∨ -1) ∧ tag != 0 ⟹ -EINVAL. |
| `kbuf_only_kbuf_unregister` | INVARIANT | per-buffer_unregister_bvec: !IO_REGBUF_F_KBUF ⟹ -EBUSY. |
| `lock_two_rings_pointer_order` | INVARIANT | per-clone: ctx1 < ctx2 always locked first. |

### Layer 2: TLA+

`models/io_uring/rsrc-update.tla`:
- Per-register/unregister/update lifecycle: states {Empty, Registered, Updating, Cloning, Replacing, Freeing}.
- Properties:
  - `safety_no_double_free` — per-imu: refcount_dec_and_test ⟹ exactly one io_free_imu.
  - `safety_no_leak_after_unregister` — per-unregister: every node either freed or transferred to clone.
  - `safety_accounting_conservation` — per-register/unregister: cumulative locked_vm/pinned_vm balances to zero.
  - `safety_clone_no_reaccount` — per-clone: cloned imus do not increment acct_pages on user.
  - `safety_files_update_skip` — per-skip sentinel: slot value preserved.
  - `liveness_register_terminates` — per-register: success ∨ rollback in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Rsrc::sqe_files_register` post: |nodes| ≤ nr_args; sparse-slot tag rejected | `Rsrc::sqe_files_register` |
| `Rsrc::sqe_buffer_register` post: imu.refs == 1 ∧ acct_pages charged | `Rsrc::sqe_buffer_register` |
| `Rsrc::clone_buffers` post: shared imu refcounts ↑; src + dst point to same imu | `Rsrc::clone_buffers` |
| `Rsrc::import_fixed` post: iter range ⊆ imu.ubuf..ubuf+len ∧ ddir compatible | `Rsrc::import_fixed` |
| `Rsrc::buffer_register_bvec` post: imu.flags & IO_REGBUF_F_KBUF | `Rsrc::buffer_register_bvec` |
| `Rsrc::files_update_with_index_alloc` post: each returned slot is a fresh fixed-file index | `Rsrc::files_update_with_index_alloc` |
| `Rsrc::free_rsrc_node` post: file fput XOR buffer_unmap; tag CQE emitted | `Rsrc::free_rsrc_node` |
| `Rsrc::register_clone_buffers` post: src and dst ctxs hold uring_lock in pointer order | `Rsrc::register_clone_buffers` |

### Layer 4: Verus/Creusot functional

`Per-IORING_REGISTER_BUFFERS → io_pin_pages → coalesce → bvec → RLIMIT_MEMLOCK+pinned_vm charge → ctx.buf_table[i].buf = imu` semantic equivalence;
`Per-IORING_REGISTER_FILES_UPDATE → array_index_nospec → reset_rsrc_node → io_fixed_file_set → bitmap update` semantic equivalence;
`Per-IORING_REGISTER_CLONE_BUFFERS → lock_two_rings → refcount_inc(src.imu) → DST_REPLACE-free → install` semantic equivalence; per Documentation/io_uring/register.rst and the liburing register test suite.

### hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

io_uring/rsrc reinforcement:

- **Per-io_is_uring_fops reject in fixed-file registration** — defense against per-recursive-uring nesting / refcount cycle.
- **Per-RLIMIT_MEMLOCK + mm.pinned_vm accounting** — defense against per-unbounded-page-pin DoS.
- **Per-IORING_MAX_FIXED_FILES (1<<20) cap** — defense against per-table-oversize OOM at register time.
- **Per-IORING_MAX_REG_BUFFERS (1<<14) cap, fits u16 buf_index** — defense against per-out-of-band buf_index encoding bug.
- **Per-array_index_nospec on every buf_index / file_index** — defense against per-Spectre v1 speculative OOB.
- **Per-validate_user_buf_range (≤ 1GiB, !overflow)** — defense against per-tiny-but-aliased-huge-len bypass.
- **Per-direction bitmap on imu (IO_IMU_DEST/_SOURCE)** — defense against per-read-via-write-only-buf info leak.
- **Per-IO_REGBUF_F_KBUF mutual exclusion in unregister_bvec** — defense against per-user vs kernel-side slot mixup.
- **Per-clone refcount_inc on imu without re-account** — defense against per-double-RLIMIT-charge.
- **Per-lock_two_rings pointer-ordered acquire** — defense against per-ABBA deadlock on cross-ring clone.
- **Per-submitter_task ownership check on clone src** — defense against per-cross-task buf-hijack.
- **Per-IORING_REGISTER_FILES_SKIP sentinel with !tag invariant** — defense against per-orphan-tag emission.
- **Per-aux CQE on node.tag at free** — observable release notification (no silent invalidation).
- **Per-headpage_already_acct dedup across pages + previously-registered table** — defense against per-double-RLIMIT-charge on aliased huge pages.

