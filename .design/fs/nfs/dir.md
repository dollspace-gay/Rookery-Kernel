# Tier-3: fs/nfs/dir.c — NFS directory operations + readdir/readdirplus + intent-based lookup

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/nfs/00-overview.md
upstream-paths:
  - fs/nfs/dir.c (~3429 lines)
  - fs/nfs/delegation.c (referenced for dir-delegation hooks)
  - fs/nfs/inode.c (referenced for nfs_set_cache_invalid + i_op wiring)
  - fs/nfs/nfs4proc.c (nfs4_dir_inode_operations + nfs4_atomic_open + nfs4_lookup_revalidate)
  - include/linux/nfs_fs.h (NFS_INO_INVALID_DATA, NFS_INO_REVAL_FORCED, nfs_inode)
  - include/linux/nfs_xdr.h (struct nfs_entry, struct nfs_readdir_arg/res)
-->

## Summary

NFS directory operations: per-directory open keeps a `struct nfs_open_dir_context` on `file->private_data` tracking the streaming cookie + `cookieverf` + cache-hit/miss counters; per-`nfs_readdir()` walks page-cache folios of decoded `struct nfs_cache_array` blobs; per-cold-folio fills via `nfs_readdir_xdr_to_array()` issuing READDIR or READDIRPLUS based on heuristic `nfs_use_readdirplus()`; per-READDIRPLUS hot-path also primes the dcache via `nfs_prime_dcache()` so subsequent `nfs_lookup()` hits without an extra LOOKUP RPC. Per-`nfs_lookup()` consults the server only when local dentry revalidation (`nfs_lookup_revalidate()`/`nfs4_lookup_revalidate()`) fails or no positive dentry exists. Per-`nfs_atomic_open()` collapses LOOKUP+OPEN+CREATE into a single NFSv4 OPEN compound (`NFS_PROTO(dir)->open_context`), which both yields a stateid and avoids a TOCTOU race against intent flags `LOOKUP_OPEN | LOOKUP_CREATE | LOOKUP_EXCL`. Per-async-invalidation: server-side directory mutations propagate through `nfs_set_cache_invalid(dir, NFS_INO_INVALID_DATA | NFS_INO_REVAL_FORCED)` from CB_NOTIFY / readdir-cookie-staleness / change-attribute mismatch. Per-directory delegations (NFSv4.1+) suppress revalidation entirely while the delegation is held — `nfs_have_directory_delegation()` short-circuits `nfs_lookup_revalidate_delegated()`. Critical for: scalable directory listing, atomic open semantics, dcache coherence with the server, READDIRPLUS-driven attribute pre-warming.

This Tier-3 covers `fs/nfs/dir.c` (~3429 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `const struct file_operations nfs_dir_operations` | per-directory file_operations | `NfsDir::FILE_OPS` |
| `const struct address_space_operations nfs_dir_aops` | per-folio cache aops | `NfsDir::ADDR_OPS` |
| `const struct dentry_operations nfs_dentry_operations` | per-NFS-dentry ops (v2/v3) | `NfsDir::DENTRY_OPS` |
| `const struct dentry_operations nfs4_dentry_operations` | per-NFSv4 dentry ops | `NfsDir::DENTRY_OPS_V4` |
| `struct inode_operations nfs4_dir_inode_operations` (nfs4proc.c) | per-NFSv4 dir inode_operations | `NfsDir::INODE_OPS_V4` |
| `struct nfs_open_dir_context` | per-fd directory iteration state | `OpenDirContext` |
| `struct nfs_cache_array_entry` | per-cached-dirent | `CacheArrayEntry` |
| `struct nfs_cache_array` | per-folio dirent batch | `CacheArray` |
| `struct nfs_readdir_descriptor` | per-readdir transient state | `ReaddirDescriptor` |
| `nfs_opendir()` / `nfs_closedir()` | per-open lifecycle | `NfsDir::open` / `close` |
| `nfs_readdir()` | per-iterate_shared | `NfsDir::readdir` |
| `nfs_readdir_xdr_to_array()` | per-folio fill | `NfsDir::readdir_xdr_to_array` |
| `nfs_readdir_xdr_filler()` | per-RPC READDIR(plus) issue | `NfsDir::xdr_filler` |
| `nfs_readdir_folio_filler()` | per-folio XDR-decode loop | `NfsDir::folio_filler` |
| `nfs_use_readdirplus()` | per-heuristic plus-vs-vanilla | `NfsDir::use_readdirplus` |
| `nfs_prime_dcache()` | per-entry dcache prime (RDPLUS) | `NfsDir::prime_dcache` |
| `nfs_do_filldir()` | per-batch emit to userspace | `NfsDir::do_filldir` |
| `nfs_llseek_dir()` | per-seek-cookie | `NfsDir::llseek` |
| `nfs_lookup()` | per-LOOKUP-RPC | `NfsDir::lookup` |
| `nfs_lookup_revalidate()` / `nfs4_lookup_revalidate()` | per-d_revalidate | `NfsDir::lookup_revalidate` / `lookup_revalidate_v4` |
| `nfs_check_verifier()` | per-verifier compare | `NfsDir::check_verifier` |
| `nfs_force_lookup_revalidate()` | per-dir cache_change_attribute bump | `NfsDir::force_lookup_revalidate` |
| `nfs_set_verifier()` | per-dentry verifier set | `NfsDir::set_verifier` |
| `nfs_atomic_open()` | per-LOOKUP+OPEN(+CREATE) compound | `NfsDir::atomic_open` |
| `nfs_atomic_open_v23()` | per-v2/v3 (no compound) emulation | `NfsDir::atomic_open_v23` |
| `nfs_create()` / `nfs_mknod()` / `nfs_mkdir()` / `nfs_rmdir()` / `nfs_unlink()` / `nfs_symlink()` / `nfs_link()` / `nfs_rename()` | per-VFS mutator | `NfsDir::create` / ... |
| `nfs_permission()` | per-access-check (cached or AUTH RPC) | `NfsDir::permission` |
| `nfs_access_get_cached()` / `nfs_access_add_cache()` | per-access-cache | `AccessCache::get` / `add` |
| `nfs_access_zap_cache()` | per-invalidate | `AccessCache::zap` |
| `nfs_dentry_delete()` / `nfs_dentry_iput()` / `nfs_d_release()` | per-dentry teardown | `NfsDir::dentry_*` |
| `nfs_set_cache_invalid()` (NFS_INO_INVALID_DATA) | per-async dir invalidate | shared (inode.c) |

## Compatibility contract

REQ-1: const struct file_operations nfs_dir_operations:
- .llseek = nfs_llseek_dir.
- .read = generic_read_dir (-EISDIR for read on dir).
- .iterate_shared = nfs_readdir.
- .open = nfs_opendir.
- .release = nfs_closedir.
- .fsync = nfs_fsync_dir (no-op; all dir ops synchronous server-side).

REQ-2: const struct address_space_operations nfs_dir_aops:
- .free_folio = nfs_readdir_clear_array — per-folio reclaim drops cached dirent name strings.

REQ-3: const struct dentry_operations nfs_dentry_operations (v2/v3):
- .d_revalidate = nfs_lookup_revalidate.
- .d_weak_revalidate = nfs_weak_revalidate.
- .d_delete = nfs_dentry_delete.
- .d_iput = nfs_dentry_iput.
- .d_automount = nfs_d_automount.
- .d_release = nfs_d_release.

REQ-4: const struct dentry_operations nfs4_dentry_operations (v4):
- .d_revalidate = nfs4_lookup_revalidate (intent-aware: LOOKUP_OPEN defers to f_op->open).
- rest as REQ-3.

REQ-5: struct nfs_open_dir_context:
- list (rcu) ∈ nfsi->open_files.
- attr_gencount: per-snapshot generation when opened.
- dir_cookie: per-NFS cookie for next entry (verifier-tied).
- last_cookie: per-last successfully emitted.
- page_index: per-folio index in the dir mapping.
- dtsize: per-tuned READDIR transfer size (NFS_INIT_DTSIZE = 64 KiB initial).
- cache_hits / cache_misses: per-atomic counters feeding readdirplus heuristic.
- force_clear, eof.
- verf[NFS_DIR_VERIFIER_SIZE]: per-cookieverf snapshot.
- list_add_tail_rcu under dir->i_lock; first opener possibly triggers NFS_INO_DATA_INVAL_DEFER -> NFS_INO_INVALID_DATA | REVAL_FORCED on the dir.

REQ-6: struct nfs_cache_array (page-cache-folio payload):
- change_attr: per-folio NFS change_attr at fill time.
- last_cookie: per-folio terminal cookie (next-folio key).
- size: number of valid entries (counted_by).
- folio_full, folio_is_eof, cookies_are_ordered: per-folio flags.
- array[]: nfs_cache_array_entry { cookie, ino, name (kmemdup'd), name_len, d_type }.

REQ-7: nfs_opendir(dir, filp):
- nfs_inc_stats(dir, NFSIOS_VFSOPEN).
- ctx = alloc_nfs_open_dir_context(dir): kzalloc + attr_gencount snapshot + dtsize-init + verf-copy + list_add_tail_rcu under dir->i_lock.
- Per-first-open-after-deferred-invalidate: if open_files was empty ∧ (cache_validity & NFS_INO_DATA_INVAL_DEFER): nfs_set_cache_invalid(dir, NFS_INO_INVALID_DATA | NFS_INO_REVAL_FORCED).
- filp->private_data = ctx.

REQ-8: nfs_closedir(inode, filp):
- put_nfs_open_dir_context(file_inode(filp), filp->private_data).
- list_del_rcu under dir->i_lock; kfree_rcu.

REQ-9: nfs_readdir(file, ctx):
- nfs_inc_stats(NFSIOS_VFSGETDENTS).
- /* Invalidate-on-server-change */
- nfs_revalidate_mapping(inode, file->f_mapping).
- Allocate transient nfs_readdir_descriptor; load snapshot from dir_ctx under file->f_lock; cache_hits/cache_misses = atomic_xchg(0).
- if desc->eof: return 0.
- desc->plus = nfs_use_readdirplus(inode, ctx, cache_hits, cache_misses).
- desc->clear_cache = nfs_readdir_handle_cache_misses(inode, desc, cache_misses, force_clear).
- Loop:
  - res = readdir_search_pagecache(desc) /* find-or-fill page-cache folio matching cookie */.
  - if -EBADCOOKIE: uncached_readdir(desc) /* server lost cookie or end-of-dir */.
  - if -ETOOSMALL ∧ desc->plus: nfs_zap_caches; desc->plus = false; retry.
  - nfs_do_filldir(desc, cookieverf) /* emit to user via desc->ctx->actor */.
  - if folio_index == folio_index_max: desc->clear_cache = force_clear.
- Until desc->eob (user-buffer-full) ∨ desc->eof.
- Per-writeback to dir_ctx under file->f_lock.

REQ-10: nfs_readdir_xdr_to_array(desc, verf, pages, folio):
- /* Allocate scratch pages, issue READDIR/PLUS, decode into nfs_cache_array */
- pages = nfs_readdir_alloc_pages(npages).
- nfs_readdir_xdr_filler(desc, verf, cookie, pages, bufsize, verf_res):
  - struct nfs_readdir_arg { dentry, cred = filp->f_cred, verf, cookie, pages, page_len, plus }.
  - error = NFS_PROTO(dir)->readdir(&arg, &res) /* dispatch v2/v3/v4 */.
  - Per-server-doesnt-support-READDIRPLUS: error == -ENOTSUPP ∧ desc->plus -> clear NFS_CAP_READDIRPLUS ∧ desc->plus = false; retry once.
- nfs_readdir_folio_filler: xdr_init_decode on the buffer; loop nfs_readdir_entry_decode -> nfs_readdir_folio_array_append + (per-plus) nfs_prime_dcache.
- Sets folio_full / folio_is_eof / cookies_are_ordered flags.
- Per-cookieverf mismatch on next folio: caller drops folio and re-reads from cookie zero.

REQ-11: nfs_use_readdirplus(dir, ctx, cache_hits, cache_misses):
- if !nfs_server_capable(dir, NFS_CAP_READDIRPLUS): false.
- if NFS_SERVER(dir)->flags & NFS_MOUNT_FORCE_RDIRPLUS: true.
- if ctx->pos == 0 ∨ cache_hits + cache_misses > NFS_READDIR_CACHE_USAGE_THRESHOLD (8): true /* first batch or "ls -l" detected */.
- else: false.

REQ-12: nfs_prime_dcache(parent, entry, dir_verifier) (per-PLUS-entry):
- if !(entry->fattr->valid & NFS_ATTR_FATTR_FILEID): return.
- if !(entry->fattr->valid & NFS_ATTR_FATTR_FSID): return.
- Reject zero-len, embedded NUL, '/', "." / "..".
- filename.hash = full_name_hash(parent, name, len).
- dentry = d_lookup(parent, &filename); else d_alloc_parallel.
- if !d_in_lookup(dentry):
  - if !nfs_fsid_equal(sb_fsid, fattr_fsid): out (mountpoint).
  - if nfs_same_file(dentry, entry): nfs_set_verifier(dentry, dir_verifier); nfs_refresh_inode; nfs_setsecurity.
  - else: d_invalidate.
- else (in-lookup): inode = nfs_fhget(parent->d_sb, fh, fattr); d_splice_alias.

REQ-13: nfs_do_filldir(desc, cookieverf):
- For each nfs_cache_array_entry from cache_entry_index until eob/eof:
  - if !desc->plus ∨ ... pass d_type + ino to desc->ctx->actor.
  - Track filldir EOB return: desc->eob = true.

REQ-14: nfs_llseek_dir(filp, offset, whence):
- SEEK_SET only with offset >= 0; SEEK_CUR; default -EINVAL (SEEK_END not supported).
- Per-changed-offset: filp->f_pos = offset; dir_ctx->page_index = 0; reset dir_cookie/last_cookie to 0 (or to offset when nfs_readdir_use_cookie() true — 64-bit pos = NFS cookie); dir_ctx->eof = false.

REQ-15: nfs_force_lookup_revalidate(dir) (caller holds dir->i_lock):
- NFS_I(dir)->cache_change_attribute += 2 (bit 0 reserved for delegation tag).
- Exported; called from delegation-recall + atomic_open failure + mount + nfs_rename.

REQ-16: nfs_check_verifier(dir, dentry, rcu_walk) -> bool:
- /* "true" iff dentry's stored verifier still matches dir's current cache_change_attribute */.
- Returns false to force lookup_revalidate to issue an RPC.

REQ-17: nfs_lookup(dir, dentry, flags):
- if dentry->d_name.len > NFS_SERVER(dir)->namelen: -ENAMETOOLONG.
- if S_ISDIR ∧ !nfs_server_capable(NFS_CAP_CASE_INSENSITIVE) ∧ no LOOKUP_EXCL|LOOKUP_PARENT|LOOKUP_REVAL: nfs_readdir_record_entry_cache_miss(dir) (signals "ls -l" pattern; future readdir prefers PLUS).
- fhandle, fattr = local-alloc.
- NFS_PROTO(dir)->lookup(dir, dentry, fhandle, fattr) (LOOKUP RPC).
- Per--ENOENT: dir_verifier = nfs_save_change_attribute(dir) or inode_peek_iversion_raw (case-insensitive); d_splice_alias(NULL, dentry); nfs_set_verifier.
- inode = nfs_fhget(dir->i_sb, fhandle, fattr).
- d_splice_alias(inode, dentry).

REQ-18: nfs4_lookup_revalidate(dir, name, dentry, flags) (NFSv4):
- if __nfs_lookup_revalidate fails RCU-walk: -ECHILD.
- if !(flags & LOOKUP_OPEN) ∨ (flags & LOOKUP_DIRECTORY): full_reval (nfs_do_lookup_revalidate).
- if d_mountpoint: full_reval.
- inode = d_inode(dentry); if NULL: full_reval (negative-OPEN must reach atomic_open).
- if nfs_verifier_is_delegated(dentry) ∨ nfs_have_directory_delegation(inode): nfs_lookup_revalidate_delegated (skip RPC).
- if !S_ISREG: full_reval.
- if flags & (LOOKUP_EXCL | LOOKUP_REVAL): reval_dentry (force nfs_lookup_revalidate_dentry).
- if !nfs_check_verifier(dir, dentry, LOOKUP_RCU): reval_dentry.
- Else: return 1 — let f_op->open() perform OPEN-as-revalidate (the open compound includes attrs).

REQ-19: nfs_atomic_open(dir, dentry, file, open_flags, mode) (NFSv4):
- BUG_ON(d_inode(dentry)) — must be called on negative dentry.
- nfs_check_flags(open_flags).
- if O_DIRECTORY: lookup_flags = LOOKUP_OPEN|LOOKUP_DIRECTORY; goto no_open (NFS server cannot OPEN a dir).
- if len > namelen: -ENAMETOOLONG.
- if O_CREAT:
  - if !(server->attr_bitmask[2] & FATTR4_WORD2_MODE_UMASK): mode &= ~current_umask().
  - attr.ia_valid |= ATTR_MODE; attr.ia_mode = mode.
- if O_TRUNC: attr.ia_valid |= ATTR_SIZE; attr.ia_size = 0.
- if !O_CREAT ∧ !d_in_lookup: d_drop; switched=true; d_alloc_parallel.
- ctx = alloc_nfs_open_context(dentry, flags_to_mode(open_flags), file).
- inode = NFS_PROTO(dir)->open_context(dir, ctx, open_flags, &attr, &created) /* OPEN compound: PUTFH+OPEN+GETFH+GETATTR */.
- if created: file->f_mode |= FMODE_CREATED.
- Per-error -ENOENT: dir_verifier = case-insensitive ? inode_peek_iversion_raw : nfs_save_change_attribute; nfs_set_verifier; d_splice_alias(NULL, dentry).
- Per-error -EISDIR/-ENOTDIR/-ELOOP-without-NOFOLLOW: goto no_open (fall back to LOOKUP-only).
- file->f_mode |= FMODE_CAN_ODIRECT.
- err = nfs_finish_open(ctx, ctx->dentry, file, open_flags) -> finish_open + (S_ISREG) nfs_file_set_open_context.

REQ-20: nfs_atomic_open_v23(dir, dentry, file, open_flags, mode) (v2/v3 fallback):
- No OPEN compound; emulate via nfs_do_create then finish_open.
- O_CREAT: nfs_do_create(dir, dentry, mode, open_flags); if !error: file->f_mode |= FMODE_CREATED; finish_open.
- Else: lookup-only path.

REQ-21: Directory write-delegation (NFSv4.1+) (delegation.c hooks, used by dir.c):
- nfs_have_directory_delegation(dir) — client holds write-style delegation on the dir.
- nfs_test_verifier_delegated / nfs_set_verifier_delegated — encode "verifier from delegation" via bit 0 of verifier; trusted without RPC re-check.
- nfs_lookup_revalidate_delegated bypasses the LOOKUP RPC while delegation is held.
- Delegation recall (CB_RECALL_ANY / CB_NOTIFY): nfs_clear_verifier_delegated drops the bit-0 trust marker; future revalidations re-engage RPC path.

REQ-22: Async invalidation:
- Server signal sources: stale cookie (READDIR returns NFS4ERR_BAD_COOKIE / NFS3ERR_BAD_COOKIE), change_attr mismatch on parent, CB_NOTIFY, CREATE/REMOVE/RENAME completion locally.
- Marker: nfs_set_cache_invalid(dir, NFS_INO_INVALID_DATA | NFS_INO_REVAL_FORCED) — set in inode.c.
- Result: next nfs_readdir → nfs_revalidate_mapping triggers nfs_invalidate_mapping → invalidate_inode_pages2 frees folios → nfs_readdir_clear_array via .free_folio drops cached names.
- Per-NFS_INO_DATA_INVAL_DEFER: deferred-until-first-open marker used when no opener present (skips eager invalidate).

REQ-23: nfs_access permission cache (LRU shrinker-managed):
- struct nfs_access_entry { cred, mask, timestamp, rb_node, lru } per (inode, cred) pair.
- nfs_access_get_cached_rcu fast-path under RCU; rb-search by cred-cmp.
- Per-RCU-fail: nfs_access_get_cached_locked under inode->i_lock; per-stale: nfs_access_zap_cache.
- nfs_access_cache_scan / count: shrinker callbacks, capped by nfs_access_max_cachesize (default 4 MiB).
- Per-eviction: cred drop + rb_erase + kfree.

REQ-24: nfs_permission(idmap, inode, mask):
- if MAY_NOT_BLOCK ∧ ¬cached: -ECHILD (fall back to slow-path).
- nfs_do_access: lookup cached → if hit ∧ matches: return.
- Else: NFS_PROTO(inode)->access(inode, &cache_entry); nfs_access_calc_mask; nfs_access_add_cache.

REQ-25: nfs_create / nfs_mknod / nfs_mkdir / nfs_rmdir / nfs_unlink / nfs_symlink / nfs_link / nfs_rename:
- All call nfs_dentry_handle_enoent on -ENOENT to convert to negative dentry.
- All adjust dir's change_attribute via nfs_set_cache_invalid(dir, NFS_INO_INVALID_DATA | ...).
- nfs_safe_remove handles open-but-unlinked → silly-rename (.nfsXXXXX).
- nfs_rename: if old_dentry has pending writeback, wait; cross-dir checks; silly-rename for cross-dir conflicts.

REQ-26: Per-stats / tracing:
- nfs_inc_stats per VFS op (NFSIOS_VFSOPEN, VFSGETDENTS, VFSLOOKUP, VFSACCESS, VFSFSYNC).
- trace_nfs_readdir_*, trace_nfs_atomic_open_enter/exit, trace_nfs_lookup_revalidate_*.

REQ-27: Per-32-bit-loff_t-safe cookies:
- is_32bit_api() detection; nfs_readdir_use_cookie returns false on 32-bit so offsets encode entry-index, not raw 64-bit cookie.

## Acceptance Criteria

- [ ] AC-1: nfs_dir_operations.iterate_shared dispatches to nfs_readdir.
- [ ] AC-2: First nfs_readdir on a fresh open issues READDIRPLUS (cache_hits=0, ctx->pos=0 first-batch heuristic).
- [ ] AC-3: After 8+ getattr-cache events, nfs_use_readdirplus returns true and subsequent fills use READDIRPLUS.
- [ ] AC-4: nfs_prime_dcache populates a positive dentry from READDIRPLUS so subsequent nfs_lookup skips LOOKUP RPC.
- [ ] AC-5: nfs_atomic_open with O_CREAT|O_EXCL on an existing file returns -EEXIST in the OPEN compound — no separate LOOKUP RPC required.
- [ ] AC-6: nfs_atomic_open with O_DIRECTORY falls back to nfs_lookup (no-OPEN-of-dir).
- [ ] AC-7: nfs_force_lookup_revalidate(dir) increments NFS_I(dir)->cache_change_attribute by 2, causing nfs_check_verifier to return false on stored-verifier-stale dentries.
- [ ] AC-8: NFS_INO_INVALID_DATA | NFS_INO_REVAL_FORCED on dir causes invalidate_inode_pages2 → nfs_readdir_clear_array drops cached array entries.
- [ ] AC-9: NFSv4 directory delegation: nfs_lookup_revalidate_delegated returns 1 without issuing a LOOKUP RPC.
- [ ] AC-10: -ETOOSMALL with desc->plus disables PLUS and retries READDIR (server quirk path).
- [ ] AC-11: nfs_readdir_xdr_filler retries once on -ENOTSUPP for READDIRPLUS, clearing NFS_CAP_READDIRPLUS.
- [ ] AC-12: nfs_llseek_dir with SEEK_END returns -EINVAL.
- [ ] AC-13: nfs_permission cached hit returns without an ACCESS RPC; cached miss issues NFS_PROTO->access exactly once.
- [ ] AC-14: nfs_access shrinker bounds cache at nfs_access_max_cachesize entries.
- [ ] AC-15: nfs_unlink of an open file does silly-rename instead of immediate REMOVE.

## Architecture

```
struct OpenDirContext {
  list: RcuListNode,                     // ∈ nfsi.open_files
  attr_gencount: AtomicU64,
  dir_cookie: u64,                       // cursor: next-entry cookie
  last_cookie: u64,
  page_index: PgOff,                     // pagecache offset of "current" folio
  dtsize: u32,                           // transfer size, doubled/halved adaptively
  cache_hits: AtomicU32,
  cache_misses: AtomicU32,
  force_clear: bool,
  eof: bool,
  verf: [u32; NFS_DIR_VERIFIER_SIZE],    // cookieverf snapshot
}

struct CacheArrayEntry {
  cookie: u64,
  ino: u64,
  name: OwnedStr,                        // kmemdup'd from XDR-decode
  name_len: u32,
  d_type: u8,
}

struct CacheArray {
  change_attr: u64,
  last_cookie: u64,
  size: u32,                             // __counted_by(size)
  folio_full: bool,
  folio_is_eof: bool,
  cookies_are_ordered: bool,
  array: Vec<CacheArrayEntry>,
}

struct ReaddirDescriptor<'f> {
  file: &'f File,
  folio: Option<&'f Folio>,
  ctx: &'f mut DirContext,
  folio_index: PgOff,
  folio_index_max: PgOff,
  dir_cookie: u64,
  last_cookie: u64,
  current_index: i64,
  verf: [u32; NFS_DIR_VERIFIER_SIZE],
  dir_verifier: u64,
  timestamp: Jiffies,
  gencount: u64,
  attr_gencount: u64,
  cache_entry_index: u32,
  buffer_fills: u32,
  dtsize: u32,
  clear_cache: bool,
  plus: bool,
  eob: bool,
  eof: bool,
}
```

`NfsDir::readdir(file, ctx) -> i32`:
1. nfs_inc_stats(inode, NFSIOS_VFSGETDENTS).
2. nfs_revalidate_mapping(inode, file->f_mapping) /* check NFS_INO_INVALID_DATA */.
3. desc = ReaddirDescriptor::zero(); desc.file = file; desc.ctx = ctx; desc.folio_index_max = !0.
4. spin_lock(&file->f_lock); copy dir_ctx -> desc; cache_hits = atomic_xchg(0); cache_misses = atomic_xchg(0); force_clear = dir_ctx.force_clear; spin_unlock.
5. if desc.eof: return 0.
6. desc.plus = NfsDir::use_readdirplus(inode, ctx, cache_hits, cache_misses).
7. force_clear = NfsDir::readdir_handle_cache_misses(inode, desc, cache_misses, force_clear).
8. desc.clear_cache = force_clear.
9. loop {
   - res = NfsDir::readdir_search_pagecache(desc).
   - match res {
     - -EBADCOOKIE: res = 0; if desc.dir_cookie != 0 ∧ !desc.eof: res = NfsDir::uncached_readdir(desc); if res == 0: continue; if res == -EBADCOOKIE ∨ -ENOTSYNC: res = 0.; break.
     - -ETOOSMALL when desc.plus: nfs_zap_caches(inode); desc.plus = false; desc.eof = false; continue.
     - other < 0: break.
     - 0: NfsDir::do_filldir(desc, nfsi.cookieverf); NfsDir::readdir_folio_unlock_and_put_cached(desc); if desc.folio_index == desc.folio_index_max: desc.clear_cache = force_clear.
   }
   - if desc.eob ∨ desc.eof: break.
10. spin_lock(&file->f_lock); writeback dir_ctx fields; spin_unlock.

`NfsDir::readdir_xdr_to_array(desc, verf, pages, folio) -> i32`:
1. pages = NfsDir::readdir_alloc_pages(npages).
2. error = NfsDir::xdr_filler(desc, verf, cookie, pages, bufsize, verf_res).
3. if error: free_pages; return error.
4. error = NfsDir::folio_filler(desc, folio, pages).
5. NfsDir::readdir_free_pages(pages, npages).

`NfsDir::xdr_filler(desc, verf, cookie, pages, bufsize, verf_res) -> i32`:
1. arg = NfsReaddirArg { dentry, cred = filp.f_cred, verf, cookie, pages, page_len = bufsize, plus = desc.plus }.
2. res = NfsReaddirRes { verf = verf_res }.
3. retry: timestamp = jiffies; gencount = nfs_inc_attr_generation_counter; desc.dir_verifier = nfs_save_change_attribute(inode).
4. error = NFS_PROTO(inode).readdir(&arg, &res).
5. if error == -ENOTSUPP ∧ desc.plus: clear NFS_CAP_READDIRPLUS; desc.plus = false; arg.plus = false; goto retry.
6. else if error < 0: return.
7. desc.timestamp = timestamp; desc.gencount = gencount.

`NfsDir::folio_filler(desc, folio, pages) -> i32`:
1. xdr_init_decode(&xdr, &xdr_buf, ...).
2. loop NfsDir::readdir_entry_decode -> NfsDir::folio_array_append:
   - entry = NfsEntry { fattr, fh, cookie, name, len, d_type, eof }.
   - if desc.plus: nfs_prime_dcache(desc.file->dentry, &entry, desc.dir_verifier).
   - if folio_array_append: full -> array.folio_full; eof -> array.folio_is_eof.

`NfsDir::use_readdirplus(dir, ctx, cache_hits, cache_misses) -> bool`:
1. if !nfs_server_capable(dir, NFS_CAP_READDIRPLUS): false.
2. if NFS_SERVER(dir).flags & NFS_MOUNT_FORCE_RDIRPLUS: true.
3. if ctx.pos == 0 ∨ cache_hits + cache_misses > NFS_READDIR_CACHE_USAGE_THRESHOLD (8): true.
4. false.

`NfsDir::prime_dcache(parent, entry, dir_verifier)`:
1. Reject !FATTR_FILEID or !FATTR_FSID.
2. Reject zero-len, NUL, '/', ".", "..".
3. dentry = d_lookup(parent, &qstr) ∨ d_alloc_parallel.
4. if d_in_lookup(dentry):
   - inode = nfs_fhget(parent.d_sb, entry.fh, entry.fattr).
   - alias = d_splice_alias(inode, dentry).
   - if alias: nfs_set_verifier(alias, dir_verifier); dput(alias).
   - d_lookup_done(dentry).
5. else:
   - if !nfs_fsid_equal(sb_fsid, fattr_fsid): out (mountpoint).
   - if nfs_same_file(dentry, entry): nfs_set_verifier(dentry, dir_verifier); nfs_refresh_inode; nfs_setsecurity.
   - else: d_invalidate(dentry).

`NfsDir::atomic_open(dir, dentry, file, open_flags, mode) -> i32`:
1. BUG_ON(d_inode(dentry)).
2. nfs_check_flags(open_flags).
3. if O_DIRECTORY:
   - if !d_in_lookup: return -ENOENT.
   - lookup_flags = LOOKUP_OPEN | LOOKUP_DIRECTORY; goto no_open.
4. if name.len > NFS_SERVER(dir).namelen: -ENAMETOOLONG.
5. if O_CREAT:
   - if !(server.attr_bitmask[2] & FATTR4_WORD2_MODE_UMASK): mode &= ~current_umask.
   - attr.ia_valid |= ATTR_MODE; attr.ia_mode = mode.
6. if O_TRUNC: attr.ia_valid |= ATTR_SIZE; attr.ia_size = 0.
7. if !O_CREAT ∧ !d_in_lookup: d_drop; switched = true; dentry = d_alloc_parallel.
8. ctx = create_nfs_open_context(dentry, open_flags, file).
9. inode = NFS_PROTO(dir).open_context(dir, ctx, open_flags, &attr, &created).
10. if created: file.f_mode |= FMODE_CREATED.
11. if IS_ERR(inode):
    - match PTR_ERR {
      - -ENOENT: dir_verifier = case-insensitive ? inode_peek_iversion_raw(dir) : nfs_save_change_attribute(dir); nfs_set_verifier; d_splice_alias(NULL, dentry).
      - -EISDIR | -ENOTDIR: goto no_open.
      - -ELOOP without O_NOFOLLOW: goto no_open.
    }
    - return PTR_ERR(inode).
12. file.f_mode |= FMODE_CAN_ODIRECT.
13. err = nfs_finish_open(ctx, ctx.dentry, file, open_flags).
14. /* no_open: fall back to LOOKUP-only via nfs_lookup */.

`NfsDir::lookup_revalidate_v4(dir, name, dentry, flags) -> i32`:
1. if __nfs_lookup_revalidate fails RCU-walk: -ECHILD.
2. if !(flags & LOOKUP_OPEN) ∨ (flags & LOOKUP_DIRECTORY): full_reval.
3. if d_mountpoint(dentry): full_reval.
4. inode = d_inode(dentry); if !inode: full_reval (negative-OPEN must hit atomic_open).
5. if nfs_verifier_is_delegated ∨ nfs_have_directory_delegation: return nfs_lookup_revalidate_delegated.
6. if !S_ISREG: full_reval.
7. if flags & (LOOKUP_EXCL | LOOKUP_REVAL): reval_dentry.
8. if !nfs_check_verifier(dir, dentry, flags & LOOKUP_RCU): reval_dentry.
9. return 1 /* let f_op->open handle revalidation */.
10. reval_dentry: if LOOKUP_RCU: -ECHILD; return nfs_lookup_revalidate_dentry(dir, name, dentry, inode, flags).
11. full_reval: return nfs_do_lookup_revalidate(dir, name, dentry, flags).

`NfsDir::force_lookup_revalidate(dir)`:
1. /* dir->i_lock held */
2. NFS_I(dir).cache_change_attribute += 2 /* bit 0 reserved for delegation tag */.

`AccessCache::get_cached_rcu(inode, cred, mask) -> i32`:
1. rcu_read_lock.
2. nfsi = NFS_I(inode).
3. e = nfs_access_search_rbtree(inode, cred).
4. if !e: rcu_read_unlock; return -ENOENT.
5. if time_after(jiffies, e.jiffies + cache_timeout): rcu_read_unlock; return -ENOENT (caller zaps).
6. *mask = e.mask.
7. rcu_read_unlock; return 0.

`AccessCache::add(inode, set, cred)`:
1. e = kmalloc; e.cred = get_cred(cred); e.mask = set.mask; e.jiffies = jiffies.
2. spin_lock(&inode.i_lock); nfs_access_add_rbtree(inode, e, cred).
3. spin_lock(&nfs_access_lru_lock); list_add_tail(&e.lru, &nfs_access_lru_list); atomic_long_inc(&nfs_access_nr_entries); spin_unlock.
4. nfs_access_cache_enforce_limit /* drop oldest if over nfs_access_max_cachesize */.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `open_dir_context_refcounted` | INVARIANT | per-opendir/closedir: ctx alloc'd before list_add_rcu; kfree_rcu only after list_del_rcu. |
| `cache_array_size_counted_by` | INVARIANT | per-cache_array: array[] length == size (counted_by attribute). |
| `prime_dcache_filename_clean` | INVARIANT | per-prime_dcache: rejects '\0', '/', "." / "..". |
| `atomic_open_negative_dentry` | INVARIANT | per-atomic_open: BUG_ON(d_inode(dentry)). |
| `force_lookup_revalidate_holds_ilock` | INVARIANT | per-force_lookup_revalidate: caller holds dir->i_lock. |
| `access_cache_cred_balanced` | INVARIANT | per-AccessCache::add/drop: get_cred ↔ put_cred 1:1. |
| `readdirplus_caps_clear_on_enotsupp` | INVARIANT | per-xdr_filler: -ENOTSUPP ∧ plus -> NFS_CAP_READDIRPLUS cleared exactly once. |
| `cookieverf_monotonic` | INVARIANT | per-readdir loop: verf re-checked across folios; mismatch -> drop folio. |

### Layer 2: TLA+

`fs/nfs/dir.tla`:
- States: per-open-fd lifecycle, per-readdir folio fill, per-RDPLUS dcache prime, per-atomic-open intent path, per-async-invalidate (CB_NOTIFY / change_attr-mismatch).
- Properties:
  - `safety_negative_dentry_into_atomic_open` — d_inode(dentry) == NULL on entry to nfs_atomic_open.
  - `safety_no_open_of_directory` — O_DIRECTORY never reaches NFS_PROTO->open_context.
  - `safety_readdirplus_capability_monotone` — once cleared, NFS_CAP_READDIRPLUS only re-set on remount.
  - `safety_dir_verifier_change_invalidates_dentries` — force_lookup_revalidate ⟹ all dentries with stored verifier fail nfs_check_verifier.
  - `safety_delegation_short_circuits_lookup` — directory delegation ⟹ no LOOKUP RPC.
  - `liveness_readdir_terminates` — bounded number of folio fills (server EOF or user EOB).
  - `liveness_atomic_open_terminates` — OPEN compound finishes ∨ falls back to lookup ∨ returns error.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `NfsDir::readdir` post: ret ≤ 0 ∨ filled-bytes ≤ user-buffer-size | `NfsDir::readdir` |
| `NfsDir::atomic_open` post: file populated ⟹ FMODE_OPENED ∨ -err | `NfsDir::atomic_open` |
| `NfsDir::xdr_filler` post: -ENOTSUPP retried ≤ 1 time | `NfsDir::xdr_filler` |
| `NfsDir::prime_dcache` post: dentry positive ⟹ inode i_ino == entry.fattr.fileid | `NfsDir::prime_dcache` |
| `NfsDir::lookup` post: dentry hashed ∧ (d_inode = result ∨ d_inode = NULL) | `NfsDir::lookup` |
| `NfsDir::permission` post: cached-hit ⟹ no RPC issued | `NfsDir::permission` |
| `AccessCache::add` post: nfs_access_nr_entries ≤ nfs_access_max_cachesize | `AccessCache::add` |

### Layer 4: Verus/Creusot functional

Per-NFSv4 OPEN compound semantics (RFC 7530 §16.16): nfs_atomic_open semantic equivalence to the (PUTFH, OPEN, GETFH, GETATTR) compound dispatched by NFS_PROTO->open_context — including O_CREAT|GUARDED4 == O_EXCL, O_CREAT|UNCHECKED4 == default, mode-stripping of umask when server lacks MODE_UMASK bitmask bit. Per-READDIRPLUS semantics (RFC 7530 §16.25 / RFC 1813 §3.3.17): entries carry both fhandle and fattr, allowing nfs_prime_dcache to manufacture a fully-cached positive dentry. Per-dcache verifier protocol: dentry's stored verifier == NFS_I(dir).cache_change_attribute at lookup time; force_lookup_revalidate's `+= 2` increments the verifier modulo bit-0 reservation for delegation tagging.

## Hardening

(Inherits row-1 features from `fs/nfs/00-overview.md` § Hardening.)

NFS dir reinforcement:

- **Per-cache_array `__counted_by(size)`** — defense against per-OOB access in cached dirent slab.
- **Per-prime_dcache filename validation** — rejects '\0', '/', ".", ".." — defense against per-server crafted directory entry (path-injection / dotdot-escape).
- **Per-O_DIRECTORY no-OPEN** — NFS server cannot OPEN a directory; explicit no_open path — defense against per-server-side stateid-on-dir UB.
- **Per-atomic_open BUG_ON(positive dentry)** — defense against per-VFS-invariant violation (atomic_open is negative-only).
- **Per-FATTR4_WORD2_MODE_UMASK capability gate** — defense against per-server-doesnt-apply-umask silent mode inflation.
- **Per-cookieverf re-check per folio** — defense against per-server-rebooted-mid-readdir cookie aliasing.
- **Per-NFS_CAP_READDIRPLUS auto-clear on -ENOTSUPP** — defense against per-buggy-server infinite RDPLUS retry.
- **Per-NFS_INO_DATA_INVAL_DEFER** — defense against per-eager-invalidate on idle dir without openers.
- **Per-access-cache shrinker (4 MiB cap default)** — defense against per-RAM-DoS via uncached-cred-storm.
- **Per-silly-rename on unlink-open** — defense against per-server rejecting REMOVE of open file (correctness, not bypass).
- **Per-d_invalidate on RDPLUS-fileid-mismatch** — defense against per-stale-positive-dentry pointing to wrong inode.
- **Per-LOOKUP_REVAL forced-RPC** — defense against per-client-side stale-cache misroute.
- **Per-32-bit-loff_t encoding via is_32bit_api** — defense against per-truncation of 64-bit NFS cookies on 32-bit userspace.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on `nfs_readdir` → `filldir` callbacks so a malicious server cannot drive an oversize copy_to_user via a crafted `READDIR` reply.
- **PAX_KERNEXEC** — W^X for any executable mapping reachable from NFS dir code paths.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization across `nfs_lookup` / `nfs_readdir` so wire-side timing cannot leak stack layout.
- **PAX_REFCOUNT** — saturating refcount on every `struct nfs_open_dir_context`, `nfs_cache_array`, and on the per-cred access-cache entry.
- **PAX_MEMORY_SANITIZE** — zero-on-free for the readdir-array pages and for the access-cache slab so stale server data does not survive into reallocation.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access on the dirent emit path.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `nfs_dir_inode_ops` / `nfs_dir_ops` and on RPC `proc.encode`/`proc.decode` vtables.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding in `/proc/self/mountstats` NFS dentry counters.
- **GRKERNSEC_DMESG** — syslog restriction on RPC-level diagnostics that leak server addresses.
- **NFSv4 `atomic_open` PAX_USERCOPY** — `nfs4_atomic_open` round-trips the `open_flags`/`mode_t` and returned `nfs4_state` under bounded copy semantics so a misbehaving server cannot trigger oversize copy into the lookup result.
- **READDIRPLUS opt-out** — `nfs_advise_use_readdirplus` defers RDPLUS once the dir is "leaf-shaped" and falls back to plain READDIR; this prevents server-side amplification attacks that try to force the client to pre-populate the inode cache.
- **`d_invalidate` on RDPLUS fileid mismatch** — a server that hands back the wrong (fh, fileid) for a known dentry forces dentry invalidation rather than silent rebinding.
- **`LOOKUP_REVAL` forced-RPC** — `O_CREAT|O_EXCL` and rename paths force a fresh GETATTR so stale-cache hides cannot win.
- **`is_32bit_api` cookie encoding** — `nfs_readdir_page_filler` collapses 64-bit cookies to 32-bit safely for 32-bit userspace consumers without losing entries.
- **Silly-rename under unlink-of-open** — `nfs_sillyrename` makes the client correct for servers that refuse `REMOVE` of an open file.
- **Access-cache shrinker (4 MiB cap)** — per-cred access cache is reclaim-bounded so an unprivileged cred-storm cannot pin server-evaluated permissions.

Per-doc rationale: NFS directory ops are the largest attack surface for server-side adversaries: a malicious or compromised server controls fileids, cookies, RDPLUS attribute streams, and access masks. PaX/grsec reinforcement turns server-driven misbehavior into bounded refusals (`-EIO`, `-ESTALE`, dentry invalidation) rather than client-side UAF, refcount wrap, or oversize copy_to_user.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `fs/nfs/inode.c` general inode lifecycle (covered in `fs/nfs/inode-dir-file.md` Tier-3)
- `fs/nfs/nfs4state.c` open-stateid state machine (covered in `fs/nfs/v4.md` Tier-3)
- `fs/nfs/delegation.c` recall + state recovery (covered in `fs/nfs/delegation.md` Tier-3)
- `fs/nfs/nfs4callback.c` CB_NOTIFY (covered in `fs/nfs/v4.md` Tier-3)
- `fs/nfs/write.c` / `fs/nfs/read.c` data path (covered in `fs/nfs/inode-dir-file.md` Tier-3)
- NFSv4 OPEN compound XDR (covered in `fs/nfs/v4.md` Tier-3)
- pNFS layout state on directory inodes (covered in `fs/nfs/pnfs.md` Tier-3)
- Implementation code
