# Tier-3: fs/fuse/inode.c — FUSE inode + super-block lifecycle

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/fuse/00-overview.md
upstream-paths:
  - fs/fuse/inode.c (~2345 lines)
  - fs/fuse/fuse_i.h
  - fs/fuse/dev.c (fuse_dev / iqueue / pqueue)
  - include/uapi/linux/fuse.h (struct fuse_attr, fuse_init_in/out, FUSE_INIT, FUSE_DESTROY, FUSE_STATFS, FUSE_SYNCFS)
  - include/uapi/linux/magic.h (FUSE_SUPER_MAGIC)
-->

## Summary

`fs/fuse/inode.c` owns FUSE per-`super_block` setup, per-`fuse_conn` allocation, per-`fuse_inode` cache, per-mount-option parsing, the `FUSE_INIT` handshake reply, and per-super-block teardown for both `fuse` (anon) and `fuseblk` (block-backed) mount types. Per-`struct fuse_conn` holds connection state shared by all mounts sharing the same `/dev/fuse` fd (refcounted, killsb rwsem, iqueue, bg/limits, max_pages, scramble_key, user_ns, pid_ns, timeout machinery, dax/passthrough optional). Per-`struct fuse_inode` is a fuse_inode_cachep-backed object embedding `struct inode` (BUILD_BUG_ON(offsetof != 0)) plus nodeid, nlookup, forget, attr_version, i_time (attr TTL), inval_mask, write/queued/iocachectr, mutex/spinlock, optional dax/backing-file state. Per-`fuse_iget` performs `iget5_locked` on nodeid hash, initializes new inode (sets S_NOATIME, S_NOCMTIME if !writeback_cache, calls fuse_init_inode for mode-dispatch), or detects stale generation (calls `fuse_make_bad`). Per-`fuse_change_attributes` (and `_common`) imports per-`struct fuse_attr` cached values (mode, uid/gid via user-ns, atime/mtime/ctime, size, blocks, blksize) and STATX_BTIME when sx present; honors writeback_cache mtime/ctime/size pinning; truncate_pagecache + invalidate_inode_pages2 on size/mtime drift. Per-mount-option parser exposes the canonical FUSE option set: `fd=` (fuse_dev_operations fd, with user-ns match), `rootmode=`, `user_id=`, `group_id=`, `default_permissions`, `allow_other`, `max_read=`, `blksize=` (fuseblk only), `source`, `subtype`. Per-`fuse_fill_super_common` initializes the super-block defaults (FUSE_SUPER_MAGIC, fuse_super_operations, fuse_xattr_handlers, SB_POSIXACL, SB_I_IMA_UNVERIFIABLE_SIGNATURE / SB_I_NOIDMAP / SB_I_NO_DATA_INTEGRITY), sets up the BDI, allocates the root inode via `fuse_get_root_inode(rootmode)` + d_make_root, installs fud, registers fc on `fuse_conn_list`. Per-`fuse_fill_super` requires fud + rootmode_present + user_id_present + group_id_present, then `fuse_send_init` issues the FUSE_INIT opcode and `process_init_reply` translates negotiated flags into per-`fuse_conn` capabilities (async_read, posix_locks, atomic_o_trunc, export_support, big_writes, dont_mask, auto_inval_data / explicit_inval_data, do_readdirplus, async_dio, writeback_cache, parallel_dirops, handle_killpriv[_v2], posix_acl, cache_symlinks, abort_err, max_pages, dax options, setxattr_ext, init_security, create_supp_group, direct_io_allow_mmap, passthrough, no_export_support, allow_idmap, io_uring, request_timeout). Per-super-block teardown: per-`fuse_kill_sb_anon` / `_blk` calls `fuse_sb_destroy` (removes fm from `fc->mounts`; last-mount triggers FUSE_DESTROY + abort_conn + wait_aborted + ctl_remove_conn) then `kill_anon_super` / `kill_block_super` and finally `fuse_mount_destroy`. Critical for: FUSE-mount semantics, FUSE protocol negotiation, FUSE attr-cache TTL coherence, FUSE namespace isolation.

This Tier-3 covers `fs/fuse/inode.c` (~2345 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct fuse_conn` | per-FUSE-connection state | `FuseConn` |
| `struct fuse_inode` | per-inode FUSE private state | `FuseInode` |
| `struct fuse_mount` | per-(super_block,fc) binding | `FuseMount` |
| `struct fuse_dev` | per-/dev/fuse fd state | `FuseDev` |
| `struct fuse_fs_context` | per-mount parse context | `FuseFsContext` |
| `struct fuse_attr` | per-inode UAPI attr | `FuseAttr` |
| `struct fuse_init_in` / `fuse_init_out` | per-FUSE_INIT handshake | `FuseInitIn` / `FuseInitOut` |
| `fuse_alloc_inode()` | per-fuse_inode_cachep alloc + init | `FuseInode::alloc` |
| `fuse_free_inode()` | per-inode free + dax/passthrough release | `FuseInode::free` |
| `fuse_evict_inode()` | per-inode evict + FORGET queue | `FuseInode::evict` |
| `fuse_change_attributes()` | per-attr-cache import (TTL refresh) | `FuseInode::change_attributes` |
| `fuse_change_attributes_common()` | per-attr import without size/inval | `FuseInode::change_attributes_common` |
| `fuse_iget()` | per-nodeid inode hash lookup + alloc | `FuseInode::iget` |
| `fuse_ilookup()` | per-fc-wide nodeid lookup | `FuseInode::ilookup` |
| `fuse_reverse_inval_inode()` | per-server-initiated cache invalidation | `FuseInode::reverse_inval` |
| `fuse_try_prune_one_inode()` | per-prune-dentries-on-forget | `FuseInode::try_prune` |
| `fuse_lock_inode()` / `_unlock_inode` | per-parallel_dirops gate | `FuseInode::lock` / `unlock` |
| `fuse_umount_begin()` | per-umount abort | `FuseSb::umount_begin` |
| `fuse_send_destroy()` | per-FUSE_DESTROY RPC | `FuseConn::send_destroy` |
| `fuse_statfs()` | per-FUSE_STATFS RPC | `FuseSb::statfs` |
| `fuse_sync_fs()` | per-FUSE_SYNCFS RPC | `FuseSb::sync_fs` |
| `fuse_sync_fs_writes()` | per-bucket-rollover write-sync | `FuseSb::sync_fs_writes` |
| `fuse_sync_bucket_alloc()` | per-write-bucket alloc | `FuseSb::sync_bucket_alloc` |
| `fuse_parse_param()` | per-mount-option parse | `FuseFsCtx::parse_param` |
| `fuse_opt_fd()` | per-fd= option (user-ns match) | `FuseFsCtx::opt_fd` |
| `fuse_show_options()` | per-/proc/mounts | `FuseSb::show_options` |
| `fuse_iqueue_init()` | per-input queue init | `FuseConn::iqueue_init` |
| `fuse_pqueue_init()` | per-processing queue init | `FuseConn::pqueue_init` |
| `fuse_conn_init()` | per-fc init + defaults | `FuseConn::init` |
| `fuse_conn_get()` / `fuse_conn_put()` | per-fc refcount | `FuseConn::get` / `put` |
| `fuse_get_root_inode()` | per-root-inode alloc (FUSE_ROOT_ID) | `FuseConn::get_root_inode` |
| `fuse_get_dentry()` / `fuse_fh_to_dentry` / `fuse_fh_to_parent` / `fuse_get_parent` / `fuse_encode_fh` | per-NFS-export ops | `FuseExport::*` |
| `fuse_fill_super_common()` | per-sb fill (sans INIT send) | `FuseSb::fill_common` |
| `fuse_fill_super()` | per-sb fill + FUSE_INIT | `FuseSb::fill` |
| `fuse_fill_super_submount()` | per-submount sb fill | `FuseSb::fill_submount` |
| `fuse_get_tree()` / `fuse_get_tree_submount()` | per-fs_context mount | `FuseFsCtx::get_tree` / `get_tree_submount` |
| `fuse_init_fs_context()` / `fuse_init_fs_context_submount()` | per-fs_context init | `FuseFsCtx::init` / `init_submount` |
| `fuse_reconfigure()` | per-remount no-op | `FuseFsCtx::reconfigure` |
| `fuse_mount_remove()` / `fuse_conn_destroy()` / `fuse_sb_destroy()` / `fuse_mount_destroy()` | per-sb teardown | `FuseSb::destroy` chain |
| `fuse_kill_sb_anon()` / `fuse_kill_sb_blk()` | per-fs_type kill_sb | `FuseFs::kill_sb_anon` / `kill_sb_blk` |
| `fuse_bdi_init()` | per-BDI setup (max_ratio 1%) | `FuseSb::bdi_init` |
| `fuse_dev_alloc()` / `fuse_dev_install()` / `fuse_dev_alloc_install()` / `fuse_dev_put()` | per-fud lifecycle | `FuseDev::*` |
| `fuse_send_init()` / `fuse_new_init()` / `process_init_reply()` / `process_init_limits()` | per-INIT handshake | `FuseConn::send_init` / chain |
| `set_request_timeout()` / `init_server_timeout()` | per-request-timeout setup | `FuseConn::timeout_*` |
| `set_global_limit()` / `sanitize_global_limit()` | per-module-param helper | `FuseConn::*global_limit` |
| `fuse_inode_init_once()` / `fuse_fs_init()` / `fuse_fs_cleanup()` / `fuse_sysfs_init()` / `fuse_sysfs_cleanup()` / `fuse_init()` / `fuse_exit()` | per-module init | `FuseModule::*` |
| `struct super_operations fuse_super_operations` | per-sb-op table | `FuseSuperOps` |
| `struct export_operations fuse_export_operations` / `_fid_operations` | per-export-op table | `FuseExportOps` / `FuseExportFidOps` |
| `struct file_system_type fuse_fs_type` / `fuseblk_fs_type` | per-fs-type registration | `FuseFsType` / `FuseBlkFsType` |

## Compatibility contract

REQ-1: struct fuse_conn fields used by inode.c:
- lock (spin), bg_lock (spin), killsb (rwsem), count (refcount), epoch (atomic) + epoch_work, blocked_waitq, iq, bg_queue, entry, devices, num_waiting, max_background, congestion_threshold, khctr, polled_files, blocked, initialized, conn_init, conn_error, connected, attr_version (atomic64), evict_ctr (atomic64), scramble_key, pid_ns, user_ns, max_pages, max_pages_limit, name_max, timeout, mounts (list), curr_bucket (RCU), dev, max_read, max_write, default_permissions, allow_other, user_id, group_id, legacy_opts_show, destroy, no_control, no_force_umount, plus capability flags: async_read, no_lock, no_flock, atomic_o_trunc, export_support, big_writes, dont_mask, auto_inval_data, explicit_inval_data, do_readdirplus, readdirplus_auto, async_dio, writeback_cache, parallel_dirops, handle_killpriv, handle_killpriv_v2, posix_acl, cache_symlinks, abort_err, setxattr_ext, init_security, create_supp_group, direct_io_allow_mmap, passthrough, max_stack_depth, io_uring, plus DAX/passthrough optional fields.

REQ-2: struct fuse_inode invariants:
- offsetof(struct fuse_inode, inode) == 0 (BUILD_BUG_ON).
- fields: nodeid, nlookup, forget, attr_version, i_time, inval_mask, orig_ino, orig_i_mode, cached_i_blkbits, i_btime, state, write_files, queued_writes, iocachectr, mutex, lock, submount_lookup (optional), dax (optional), backing (optional, passthrough).

REQ-3: fuse_alloc_inode(sb):
- alloc_inode_sb(sb, fuse_inode_cachep, GFP_KERNEL).
- memset((void*)fi + sizeof(fi->inode), 0, sizeof(*fi) - sizeof(fi->inode)).
- fi.inval_mask = ~0; mutex_init; spin_lock_init.
- fi.forget = fuse_alloc_forget().
- per-CONFIG_FUSE_DAX: fuse_dax_inode_alloc.
- per-CONFIG_FUSE_PASSTHROUGH: fuse_inode_backing_set(NULL).
- return &fi.inode.

REQ-4: fuse_free_inode(inode):
- mutex_destroy; kfree(fi.forget).
- per-CONFIG_FUSE_DAX: kfree(fi.dax).
- per-CONFIG_FUSE_PASSTHROUGH: fuse_backing_put.
- kmem_cache_free(fuse_inode_cachep, fi).

REQ-5: fuse_evict_inode(inode):
- WARN_ON(I_DIRTY_INODE set).
- if FUSE_IS_DAX: dax_break_layout_final.
- truncate_inode_pages_final(&inode.i_data); clear_inode.
- if SB_ACTIVE:
  - if FUSE_IS_DAX: fuse_dax_inode_cleanup.
  - if fi.nlookup: fuse_queue_forget(fc, fi.forget, fi.nodeid, fi.nlookup); fi.forget = NULL.
  - if fi.submount_lookup: fuse_cleanup_submount_lookup; fi.submount_lookup = NULL.
  - if i_nlink > 0: atomic64_inc(&fc.evict_ctr). /* race-detector vs in-flight LOOKUP */
- if S_ISREG ∧ !fuse_is_bad: WARN iocachectr / write_files / queued_writes empty.

REQ-6: fuse_change_attributes_common(inode, attr, sx, attr_valid, cache_mask, evict_ctr):
- lockdep_assert_held(&fi.lock).
- if !evict_ctr ∨ fi.attr_version ∨ evict_ctr == fc.evict_ctr: clear STATX_BASIC_STATS in inval_mask.
- fi.attr_version = atomic64_inc_return(&fc.attr_version).
- fi.i_time = attr_valid.
- i_ino = fuse_squash_ino(attr.ino); i_mode = (i_mode & S_IFMT) | (attr.mode & 07777).
- set_nlink(inode, attr.nlink).
- i_uid = make_kuid(fc.user_ns, attr.uid); i_gid = make_kgid(fc.user_ns, attr.gid).
- i_blocks = attr.blocks.
- Sanitize nsecs to [0, NSEC_PER_SEC - 1].
- inode_set_atime; if !(cache_mask & STATX_MTIME): inode_set_mtime; if !(cache_mask & STATX_CTIME): inode_set_ctime.
- if sx: sanitize btime nsec; clear STATX_BTIME in inval_mask; if STATX_BTIME: set FUSE_I_BTIME + fi.i_btime.
- cached_i_blkbits = attr.blksize ? ilog2(attr.blksize) : sb.s_blocksize_bits.
- fi.orig_i_mode = inode.i_mode; if !fc.default_permissions: i_mode &= ~S_ISVTX.
- fi.orig_ino = attr.ino; i_flags &= ~S_NOSEC.

REQ-7: fuse_change_attributes(inode, attr, sx, attr_valid, attr_version):
- spin_lock(&fi.lock).
- cache_mask = fuse_get_cache_mask(inode) /* writeback_cache ∧ S_ISREG ⟹ STATX_MTIME|CTIME|SIZE */.
- if cache_mask & STATX_SIZE: attr.size = i_size_read.
- if cache_mask & STATX_MTIME: attr.mtime/mtimensec = inode mtime.
- if cache_mask & STATX_CTIME: attr.ctime/ctimensec = inode ctime.
- if attr_version != 0 ∧ fi.attr_version > attr_version: spin_unlock; return.
- if test_bit(FUSE_I_SIZE_UNSTABLE): spin_unlock; return.
- fuse_change_attributes_common.
- oldsize = i_size; if !(cache_mask & STATX_SIZE): i_size_write(attr.size).
- spin_unlock.
- if !cache_mask ∧ S_ISREG:
  - if oldsize != attr.size: truncate_pagecache; if !fc.explicit_inval_data: inval = true.
  - elif fc.auto_inval_data ∧ mtime changed: inval = true.
  - if inval: invalidate_inode_pages2(mapping).
- per-CONFIG_FUSE_DAX: fuse_dax_dontcache(inode, attr.flags).

REQ-8: fuse_iget(sb, nodeid, generation, attr, attr_valid, attr_version, evict_ctr):
- per-auto_submounts ∧ FUSE_ATTR_SUBMOUNT ∧ S_ISDIR: new_inode (not hashed); fuse_init_inode; fi.nodeid; allocate submount_lookup; S_AUTOMOUNT; go to done.
- retry: inode = iget5_locked(sb, nodeid, fuse_inode_eq, fuse_inode_set, &nodeid).
- if I_NEW:
  - S_NOATIME.
  - if !writeback_cache ∨ !S_ISREG(attr.mode): S_NOCMTIME.
  - i_generation = generation.
  - fuse_init_inode(inode, attr, fc) /* mode-dispatch */.
- elif fuse_stale_inode(inode, generation, attr): fuse_make_bad; if !root: remove_inode_hash + iput + retry.
- fi.nlookup++.
- done: fuse_change_attributes_i(inode, attr, NULL, attr_valid, attr_version, evict_ctr).
- if I_NEW: unlock_new_inode.

REQ-9: fuse_init_inode(inode, attr, fc):
- i_mode = attr.mode & S_IFMT; i_size = attr.size; set mtime/ctime.
- per-S_ISREG: fuse_init_common + fuse_init_file_inode(attr.flags).
- per-S_ISDIR: fuse_init_dir.
- per-S_ISLNK: fuse_init_symlink.
- per-CHR/BLK/FIFO/SOCK: fuse_init_common + init_special_inode(new_decode_dev(attr.rdev)).
- else: BUG.
- if !fc.posix_acl: inode.i_acl = i_default_acl = ACL_DONT_CACHE.

REQ-10: fuse_reverse_inval_inode(fc, nodeid, offset, len):
- inode = fuse_ilookup(fc, nodeid, NULL); if !inode return -ENOENT.
- fi.attr_version = atomic64_inc_return(&fc.attr_version).
- fuse_invalidate_attr; forget_all_cached_acls.
- if offset >= 0: invalidate_inode_pages2_range(mapping, pg_start, pg_end).
- iput.

REQ-11: fuse_umount_begin(sb):
- if fc.no_force_umount: return.
- fuse_abort_conn(fc).
- if sb.s_bdev: retire_super(sb).

REQ-12: fuse_send_destroy(fm):
- if fm.fc.conn_init: opcode = FUSE_DESTROY; force = true; nocreds = true; fuse_simple_request.

REQ-13: fuse_statfs(dentry, buf):
- if !fuse_allow_current_process(fc): buf.f_type = FUSE_SUPER_MAGIC; return 0.
- opcode = FUSE_STATFS; nodeid = root-nodeid.
- err = fuse_simple_request; if !err: convert_fuse_statfs.

REQ-14: fuse_sync_fs(sb, wait):
- if !wait: return 0. /* userspace can't handle wait==0 */
- if !s_root: return 0.
- if !fc.sync_fs: return 0.
- fuse_sync_fs_writes(fc) /* bucket rollover + wait for ancestor count == 0 */.
- opcode = FUSE_SYNCFS; nodeid = root.
- err = fuse_simple_request; if -ENOSYS: fc.sync_fs = 0; err = 0.

REQ-15: enum + struct fs_parameter_spec fuse_fs_parameters:
- OPT_SOURCE ("source", string).
- OPT_FD ("fd", fd).
- OPT_ROOTMODE ("rootmode", u32oct).
- OPT_USER_ID ("user_id", uid).
- OPT_GROUP_ID ("group_id", gid).
- OPT_DEFAULT_PERMISSIONS (flag).
- OPT_ALLOW_OTHER (flag).
- OPT_MAX_READ ("max_read", u32).
- OPT_BLKSIZE ("blksize", u32) — fuseblk only.
- OPT_SUBTYPE ("subtype", string).

REQ-16: fuse_opt_fd(fsc, file):
- if file.f_op != &fuse_dev_operations: invalfc("fd is not a fuse device").
- if file.f_cred.user_ns != fsc.user_ns: invalfc("wrong user namespace").
- ctx.fud = file.private_data; refcount_inc(&ctx.fud.ref).

REQ-17: fuse_parse_param(fsc, param):
- per-FS_CONTEXT_FOR_RECONFIGURE: oldapi ⟹ ignore (back-compat); else invalfc.
- fs_parse(fuse_fs_parameters).
- OPT_SOURCE: fsc.source = string (single).
- OPT_SUBTYPE: ctx.subtype = string (single).
- OPT_FD: per-fs_value_is_file: fuse_opt_fd(file); else fget(uint_32) + fuse_opt_fd.
- OPT_ROOTMODE: fuse_valid_type(result.uint_32); ctx.rootmode = uint_32; rootmode_present = true.
- OPT_USER_ID: kuid_has_mapping(fsc.user_ns) check; ctx.user_id = kuid; user_id_present = true.
- OPT_GROUP_ID: same for kgid.
- OPT_DEFAULT_PERMISSIONS: ctx.default_permissions = true.
- OPT_ALLOW_OTHER: ctx.allow_other = true.
- OPT_MAX_READ: ctx.max_read = uint_32.
- OPT_BLKSIZE: if !ctx.is_bdev: invalfc; else ctx.blksize = uint_32.

REQ-18: fuse_show_options(m, root):
- per-fc.legacy_opts_show: user_id, group_id, default_permissions, allow_other, max_read (if != ~0), blksize (if s_bdev ∧ != FUSE_DEFAULT_BLKSIZE).
- per-CONFIG_FUSE_DAX: dax=always / never / inode.

REQ-19: fuse_conn_init(fc, fm, user_ns, fiq_ops, fiq_priv):
- memset(fc, 0, sizeof).
- spin_lock_init(&fc.lock); spin_lock_init(&fc.bg_lock); init_rwsem(&fc.killsb).
- refcount_set(&fc.count, 1); atomic_set(&fc.epoch, 1); INIT_WORK(&fc.epoch_work).
- init_waitqueue_head(&fc.blocked_waitq).
- fuse_iqueue_init(&fc.iq, fiq_ops, fiq_priv).
- INIT_LIST_HEAD(bg_queue / entry / devices).
- atomic_set(num_waiting, 0).
- max_background = FUSE_DEFAULT_MAX_BACKGROUND (12).
- congestion_threshold = 3/4 of that.
- atomic64_set(khctr, 0).
- polled_files = RB_ROOT.
- blocked = 0; initialized = 0; connected = 1.
- attr_version = 1; evict_ctr = 1.
- get_random_bytes(&fc.scramble_key).
- pid_ns = get_pid_ns(task_active_pid_ns(current)); user_ns = get_user_ns.
- max_pages = FUSE_DEFAULT_MAX_PAGES_PER_REQ; max_pages_limit = fuse_max_pages_limit; name_max = FUSE_NAME_LOW_MAX.
- per-CONFIG_FUSE_PASSTHROUGH: fuse_backing_files_init.
- INIT_LIST_HEAD(&fc.mounts); list_add(&fm.fc_entry, &fc.mounts); fm.fc = fc.

REQ-20: fuse_conn_get/put:
- get: refcount_inc.
- put: refcount_dec_and_test; if last: dax_conn_free, cancel timeout work, cancel epoch_work, iq.ops.release, put_pid_ns, kfree curr_bucket (WARN count != 1), passthrough free; call_rcu(delayed_release) which puts user_ns and invokes fc.release(fc).

REQ-21: fuse_get_root_inode(sb, mode):
- attr = { mode, ino = FUSE_ROOT_ID, nlink = 1 }.
- return fuse_iget(sb, FUSE_ROOT_ID, 0, &attr, 0, 0, 0).

REQ-22: NFS-export via export_operations:
- fuse_export_operations (default): fh_to_dentry, fh_to_parent, encode_fh, get_parent.
- fuse_export_fid_operations: encode_fh only (used when server lacks export_support).
- encode_fh: u64 nodeid (split into u32 hi/lo) + u32 generation; per-parent: 3 more u32s; type FILEID_INO64_GEN[_PARENT].
- fh_to_dentry: nodeid from raw[0:1]; generation from raw[2]; fuse_get_dentry.
- get_parent: requires fc.export_support; fuse_lookup_name(..) on dotdot.

REQ-23: struct super_operations fuse_super_operations:
- alloc_inode = fuse_alloc_inode.
- free_inode = fuse_free_inode.
- evict_inode = fuse_evict_inode.
- write_inode = fuse_write_inode.
- drop_inode = inode_just_drop.
- umount_begin = fuse_umount_begin.
- statfs = fuse_statfs.
- sync_fs = fuse_sync_fs.
- show_options = fuse_show_options.

REQ-24: process_init_limits(fc, arg):
- if arg.minor < 13: return.
- sanitize_global_limit(&max_user_bgreq / &max_user_congthresh).
- under bg_lock:
  - if arg.max_background: fc.max_background = arg.max_background; if !cap_sys_admin ∧ > max_user_bgreq: clamp.
  - if arg.congestion_threshold: fc.congestion_threshold = arg.congestion_threshold; if !cap_sys_admin ∧ > max_user_congthresh: clamp.

REQ-25: process_init_reply(fm, args, error):
- if error ∨ arg.major != FUSE_KERNEL_VERSION: ok = false.
- else:
  - process_init_limits.
  - if arg.minor >= 6:
    - flags = arg.flags | (arg.flags2 << 32) [if FUSE_INIT_EXT].
    - ra_pages = arg.max_readahead / PAGE_SIZE.
    - decode capability flags onto fc.* per REQ-1 list.
    - if arg.minor >= 17: FUSE_FLOCK_LOCKS gates fc.no_flock.
    - if FUSE_BIG_WRITES, FUSE_DONT_MASK, FUSE_AUTO_INVAL_DATA / EXPLICIT_INVAL_DATA, FUSE_DO_READDIRPLUS, FUSE_READDIRPLUS_AUTO, FUSE_ASYNC_DIO, FUSE_WRITEBACK_CACHE, FUSE_PARALLEL_DIROPS, FUSE_HANDLE_KILLPRIV, FUSE_POSIX_ACL ⟹ default_permissions + posix_acl, FUSE_CACHE_SYMLINKS, FUSE_ABORT_ERROR, FUSE_MAX_PAGES (clamp to limit), DAX (MAP_ALIGNMENT, HAS_INODE_DAX), FUSE_HANDLE_KILLPRIV_V2 (SB_NOSEC), FUSE_SETXATTR_EXT, FUSE_SECURITY_CTX, FUSE_CREATE_SUPP_GROUP, FUSE_DIRECT_IO_ALLOW_MMAP, FUSE_PASSTHROUGH (max_stack_depth bounded, mutually exclusive with WRITEBACK_CACHE), FUSE_NO_EXPORT_SUPPORT (switch to fid_operations), FUSE_ALLOW_IDMAP (requires default_permissions), FUSE_OVER_IO_URING, FUSE_REQUEST_TIMEOUT.
    - if arg.time_gran in (0, 1000000000]: sb.s_time_gran = arg.time_gran.
  - else: ra_pages = fc.max_read / PAGE_SIZE; fc.no_lock = 1; fc.no_flock = 1.
  - init_server_timeout(fc, timeout).
  - sb.s_bdi.ra_pages = min(existing, ra_pages).
  - fc.minor = arg.minor.
  - fc.max_write = (arg.minor < 5) ? 4096 : arg.max_write; clamp max to ≥ 4096.
  - fc.conn_init = 1.
- if !ok: conn_init = 0; conn_error = 1.
- fuse_set_initialized(fc); wake_up_all(&fc.blocked_waitq).

REQ-26: fuse_new_init(fm):
- ia = kzalloc with __GFP_NOFAIL.
- ia.in.major = FUSE_KERNEL_VERSION; ia.in.minor = FUSE_KERNEL_MINOR_VERSION.
- ia.in.max_readahead = sb.s_bdi.ra_pages * PAGE_SIZE.
- flags = (kernel-supported bitmask including FUSE_INIT_EXT).
- per-CONFIG_FUSE_DAX: |= MAP_ALIGNMENT (if dax), HAS_INODE_DAX (if inode-mode).
- per-fc.auto_submounts: |= FUSE_SUBMOUNTS.
- per-CONFIG_FUSE_PASSTHROUGH: |= FUSE_PASSTHROUGH.
- per-fuse_uring_enabled: |= FUSE_OVER_IO_URING.
- ia.in.flags = flags (lo); ia.in.flags2 = flags >> 32.
- args.opcode = FUSE_INIT; in_numargs = 1; out_numargs = 1; out_argvar = true; force = true; nocreds = true.

REQ-27: fuse_send_init(fm):
- ia = fuse_new_init.
- per-fc.sync_init: args.abort_on_kill = true; err = fuse_simple_request; if err>0: err=0.
- else: args.end = process_init_reply; err = fuse_simple_background(GFP_KERNEL); if !err return 0.
- process_init_reply(fm, &ia.args, err).
- return fc.conn_error ? -ENOTCONN : 0.

REQ-28: fuse_bdi_init(fc, sb):
- if sb.s_bdev: suffix = "-fuseblk"; bdi_put(s_bdi); s_bdi = noop_backing_dev_info.
- super_setup_bdi_name("%u:%u%s", MAJOR(dev), MINOR(dev), suffix).
- s_bdi.capabilities |= BDI_CAP_STRICTLIMIT.
- bdi_set_max_ratio(s_bdi, 1) /* 1% of dirty+writeback */.

REQ-29: fuse_sb_defaults(sb):
- s_magic = FUSE_SUPER_MAGIC.
- s_op = &fuse_super_operations.
- s_xattr = fuse_xattr_handlers.
- s_maxbytes = MAX_LFS_FILESIZE.
- s_time_gran = 1.
- s_export_op = &fuse_export_operations.
- s_iflags |= SB_I_IMA_UNVERIFIABLE_SIGNATURE | SB_I_NOIDMAP | SB_I_NO_DATA_INTEGRITY.
- if s_user_ns != &init_user_ns: |= SB_I_UNTRUSTED_MOUNTER.
- s_flags &= ~(SB_NOSEC | SB_I_VERSION).

REQ-30: fuse_fill_super_common(sb, ctx):
- if SB_MANDLOCK: -EINVAL.
- rcu_assign_pointer(fc.curr_bucket, fuse_sync_bucket_alloc()).
- fuse_sb_defaults.
- if ctx.is_bdev: per-CONFIG_BLOCK: sb_set_blocksize(blksize); fc.sync_fs = 1.
- else: s_blocksize = PAGE_SIZE; s_blocksize_bits = PAGE_SHIFT.
- s_subtype = ctx.subtype; ctx.subtype = NULL.
- per-CONFIG_FUSE_DAX: fuse_dax_conn_alloc.
- fc.dev = sb.s_dev; fm.sb = sb.
- fuse_bdi_init.
- if SB_POSIXACL: fc.dont_mask = 1. sb.s_flags |= SB_POSIXACL.
- fc.default_permissions = ctx.default_permissions; allow_other; user_id; group_id; legacy_opts_show; destroy; no_control; no_force_umount.
- fc.max_read = max(4096, ctx.max_read).
- root = fuse_get_root_inode(sb, ctx.rootmode); set_default_d_op(fuse_dentry_operations); root_dentry = d_make_root.
- mutex_lock(&fuse_mutex).
- if fud: per-fuse_dev_fc_get already set: -EINVAL; if fud.sync_init: fc.sync_init = 1.
- fuse_ctl_add_conn(fc) /* /sys/fs/fuse/connections registration */.
- list_add_tail(&fc.entry, &fuse_conn_list).
- sb.s_root = root_dentry.
- if fud: fuse_dev_install(fud, fc); wake_up_all(&fuse_dev_waitq).
- mutex_unlock(&fuse_mutex).

REQ-31: fuse_fill_super(sb, fsc):
- if !ctx.fud ∨ !rootmode_present ∨ !user_id_present ∨ !group_id_present: -EINVAL.
- err = fuse_fill_super_common.
- return fuse_send_init(fm).

REQ-32: fuse_get_tree(fsc):
- alloc fc + fm.
- fuse_conn_init(fc, fm, fsc.user_ns, &fuse_dev_fiq_ops, NULL).
- fc.release = fuse_free_conn.
- if ctx.is_bdev: get_tree_bdev(fsc, fuse_fill_super).
- else: if !ctx.fud: -EINVAL.
- if fuse_dev_fc_get(ctx.fud): sget_fc(fuse_test_super, fuse_set_no_super) /* reuse existing sb */; fsc.root = dget(sb.s_root).
- else: get_tree_nodev(fsc, fuse_fill_super).
- if fsc.s_fs_info: fuse_mount_destroy(fm).

REQ-33: fuse_init_fs_context(fsc):
- ctx = kzalloc fuse_fs_context.
- ctx.max_read = ~0; ctx.blksize = FUSE_DEFAULT_BLKSIZE; ctx.legacy_opts_show = true.
- per-CONFIG_BLOCK ∧ fuseblk: ctx.is_bdev = true; ctx.destroy = true.
- fsc.fs_private = ctx; fsc.ops = &fuse_context_ops.

REQ-34: fuse_init_fs_context_submount(fsc):
- fsc.ops = &fuse_context_submount_ops.

REQ-35: fuse_get_tree_submount(fsc):
- fm = kzalloc; fm.fc = fuse_conn_get(parent-fc); fsc.s_fs_info = fm.
- sb = sget_fc(NULL, set_anon_super_fc).
- fuse_fill_super_submount(sb, mp_fi).
- down_write(&fc.killsb); list_add_tail(&fm.fc_entry, &fc.mounts); up_write.
- sb.s_flags |= SB_ACTIVE; fsc.root = dget(sb.s_root).

REQ-36: fuse_fill_super_submount(sb, parent_fi):
- fuse_sb_defaults; fm.sb = sb.
- sb.s_bdi = bdi_get(parent_sb.s_bdi).
- inherit s_xattr, s_export_op, s_time_gran, s_blocksize, s_blocksize_bits, s_subtype.
- root_attr from fuse_fill_attr_from_inode(parent_fi).
- root = fuse_iget(sb, parent_fi.nodeid, 0, &root_attr, 0, 0, fuse_get_evict_ctr(fc)).
- fi = get_fuse_inode(root); fi.nlookup-- /* duplicate; don't bump */.
- set_default_d_op(&fuse_dentry_operations); sb.s_root = d_make_root.
- if parent_fi.submount_lookup: refcount_inc + fi.submount_lookup = sl.

REQ-37: super-block teardown chain:
- fuse_kill_sb_anon: fuse_sb_destroy + kill_anon_super + fuse_mount_destroy.
- fuse_kill_sb_blk: fuse_sb_destroy + kill_block_super + fuse_mount_destroy.
- fuse_sb_destroy: if s_root: last = fuse_mount_remove(fm); if last: fuse_conn_destroy(fm).
- fuse_mount_remove: down_write(&fc.killsb); list_del_init(&fm.fc_entry); last = list_empty(&fc.mounts); up_write.
- fuse_conn_destroy: if fc.destroy: fuse_send_destroy; fuse_abort_conn; fuse_wait_aborted; if list non-empty: list_del(&fc.entry); fuse_ctl_remove_conn.
- fuse_mount_destroy: fuse_conn_put(fm.fc); kfree_rcu(fm, rcu).

REQ-38: file_system_type registration:
- fuse_fs_type: name="fuse"; flags = FS_HAS_SUBTYPE | FS_USERNS_MOUNT | FS_ALLOW_IDMAP; init_fs_context = fuse_init_fs_context; parameters = fuse_fs_parameters; kill_sb = fuse_kill_sb_anon.
- fuseblk_fs_type (CONFIG_BLOCK): name="fuseblk"; flags = FS_REQUIRES_DEV | FS_HAS_SUBTYPE | FS_ALLOW_IDMAP; kill_sb = fuse_kill_sb_blk.

REQ-39: module init/exit:
- fuse_fs_init: kmem_cache_create("fuse_inode", sizeof(struct fuse_inode), 0, SLAB_HWCACHE_ALIGN|SLAB_ACCOUNT|SLAB_RECLAIM_ACCOUNT, fuse_inode_init_once); register_fuseblk + register_filesystem(fuse_fs_type) + fuse_sysctl_register.
- fuse_fs_cleanup: fuse_sysctl_unregister + unregister_filesystem(fuse_fs_type) + unregister_fuseblk + rcu_barrier + kmem_cache_destroy.
- fuse_init: pr_info; INIT_LIST_HEAD(&fuse_conn_list); fuse_fs_init + fuse_dev_init + fuse_sysfs_init + fuse_ctl_init + fuse_dentry_tree_init; sanitize_global_limit(max_user_bgreq / max_user_congthresh).
- fuse_exit: fuse_dentry_tree_cleanup + fuse_ctl_cleanup + fuse_sysfs_cleanup + fuse_fs_cleanup + fuse_dev_cleanup.

REQ-40: Timeout machinery:
- set_request_timeout(fc, timeout): fc.timeout.req_timeout = secs_to_jiffies(timeout); INIT_DELAYED_WORK(fuse_check_timeout); queue_delayed_work(system_percpu_wq, fuse_timeout_timer_freq).
- init_server_timeout(fc, timeout): bound by fuse_max_req_timeout / fuse_default_req_timeout; floor by FUSE_TIMEOUT_TIMER_FREQ.

REQ-41: FUSE_INIT negotiated capability flags applied to fuse_conn:
- async_read, no_lock, no_flock, atomic_o_trunc, export_support, big_writes, dont_mask, auto_inval_data, explicit_inval_data, do_readdirplus, readdirplus_auto, async_dio, writeback_cache, parallel_dirops, handle_killpriv, posix_acl (also forces default_permissions), cache_symlinks, abort_err, max_pages (with name_max → FUSE_NAME_MAX when >1), MAP_ALIGNMENT/HAS_INODE_DAX (CONFIG_FUSE_DAX), handle_killpriv_v2 (sb.s_flags |= SB_NOSEC), setxattr_ext, init_security, create_supp_group, direct_io_allow_mmap, passthrough (excludes writeback_cache; bounds max_stack_depth), no_export_support (switches to fuse_export_fid_operations), allow_idmap (only with default_permissions; clears SB_I_NOIDMAP), io_uring, request_timeout.

## Acceptance Criteria

- [ ] AC-1: fuse_alloc_inode returns *inode whose container fuse_inode has inval_mask = ~0 and forget allocated.
- [ ] AC-2: fuse_free_inode releases backing-file (passthrough) and dax state before freeing.
- [ ] AC-3: fuse_evict_inode queues FORGET for nlookup if SB_ACTIVE.
- [ ] AC-4: fuse_change_attributes honors writeback_cache cache_mask (pins mtime/ctime/size).
- [ ] AC-5: fuse_change_attributes truncates pagecache on size shrink (no cache_mask) and invalidates on auto_inval mtime drift.
- [ ] AC-6: fuse_iget returns existing inode incrementing nlookup; new inode passes through fuse_init_inode mode-dispatch.
- [ ] AC-7: fuse_iget detects stale generation: fuse_make_bad + (non-root) remove from hash + retry.
- [ ] AC-8: fuse_parse_param rejects fd= not pointing at fuse_dev_operations.
- [ ] AC-9: fuse_parse_param rejects fd= whose owning user_ns differs from fsc.user_ns.
- [ ] AC-10: fuse_parse_param rejects blksize= on non-fuseblk mounts.
- [ ] AC-11: fuse_fill_super requires fud, rootmode_present, user_id_present, group_id_present.
- [ ] AC-12: process_init_reply rejects major != FUSE_KERNEL_VERSION (sets conn_error, no caps).
- [ ] AC-13: process_init_reply applies FUSE_POSIX_ACL by setting both default_permissions and posix_acl.
- [ ] AC-14: process_init_reply mutually excludes FUSE_PASSTHROUGH with FUSE_WRITEBACK_CACHE.
- [ ] AC-15: process_init_reply clamps fc.max_background and fc.congestion_threshold for non-CAP_SYS_ADMIN.
- [ ] AC-16: fuse_umount_begin respects fc.no_force_umount and retires block-backed supers.
- [ ] AC-17: fuse_kill_sb_anon / _blk last-mount triggers FUSE_DESTROY when fc.destroy is set.
- [ ] AC-18: fuse_send_init flags2 carries upper-32 of flags only when FUSE_INIT_EXT was set.
- [ ] AC-19: fuse_get_root_inode allocates with FUSE_ROOT_ID and mode = ctx.rootmode.
- [ ] AC-20: fuse_sync_fs returns 0 when !wait or !sync_fs.

## Architecture

```
struct FuseConn {              // refcounted; per-/dev/fuse fd
  lock: Spinlock,
  bg_lock: Spinlock,
  killsb: RwSemaphore,
  count: Refcount,
  epoch: AtomicI32,
  epoch_work: Work,
  blocked_waitq: WaitQueue,
  iq: FuseIqueue,
  bg_queue: ListHead,
  entry: ListHead,                // fuse_conn_list
  devices: ListHead,              // list of FuseDev
  num_waiting: AtomicI32,
  max_background: u32,
  congestion_threshold: u32,
  khctr: AtomicI64,
  polled_files: RbRoot,
  blocked: bool,
  initialized: bool,
  connected: bool,
  conn_init: bool,
  conn_error: bool,
  attr_version: AtomicI64,
  evict_ctr: AtomicI64,
  scramble_key: [u8; 32],
  pid_ns: *PidNamespace,
  user_ns: *UserNamespace,
  dev: dev_t,
  max_read: u32,
  max_write: u32,
  max_pages: u32,
  max_pages_limit: u32,
  name_max: u32,
  timeout: FuseTimeout,
  mounts: ListHead,               // list of FuseMount
  curr_bucket: Rcu<*FuseSyncBucket>,
  /* capability flags (set by process_init_reply) */
  default_permissions: bool, allow_other: bool, user_id: KUid, group_id: KGid,
  legacy_opts_show: bool, destroy: bool, no_control: bool, no_force_umount: bool,
  sync_fs: bool, sync_init: bool, abort_err: bool,
  async_read: bool, no_lock: bool, no_flock: bool, atomic_o_trunc: bool,
  export_support: bool, big_writes: bool, dont_mask: bool,
  auto_inval_data: bool, explicit_inval_data: bool,
  do_readdirplus: bool, readdirplus_auto: bool, async_dio: bool,
  writeback_cache: bool, parallel_dirops: bool,
  handle_killpriv: bool, handle_killpriv_v2: bool, posix_acl: bool,
  cache_symlinks: bool, setxattr_ext: bool, init_security: bool,
  create_supp_group: bool, direct_io_allow_mmap: bool,
  passthrough: bool, max_stack_depth: u32,
  io_uring: bool,
  /* DAX (optional) */
  dax: Option<*FuseDaxConn>, dax_mode: u8, inode_dax: bool,
  /* passthrough (optional) */
  backing_files: Option<FuseBackingFiles>,
  minor: u32,
  release: fn(*FuseConn),
  rcu: RcuHead,
}

struct FuseInode {              // BUILD_BUG_ON(offsetof(.inode) != 0)
  inode: Inode,                  // must be first
  nodeid: u64,
  nlookup: u64,
  forget: *FuseForgetLink,
  attr_version: u64,
  i_time: u64,                   // attr-cache TTL (jiffies)
  inval_mask: u32,
  orig_ino: u64,
  orig_i_mode: u16,
  cached_i_blkbits: u8,
  i_btime: Timespec64,
  state: u32,                    // FUSE_I_* flag bits
  write_files: ListHead,
  queued_writes: ListHead,
  iocachectr: AtomicI32,
  mutex: Mutex,
  lock: Spinlock,
  submount_lookup: Option<*FuseSubmountLookup>,
  dax: Option<*FuseInodeDax>,
  backing: Option<*FuseBackingFile>,
}

struct FuseMount { sb: *SuperBlock, fc: *FuseConn, fc_entry: ListHead, rcu: RcuHead, }

struct FuseFsContext {
  fud: Option<*FuseDev>,
  rootmode: u32, rootmode_present: bool,
  user_id: KUid, user_id_present: bool,
  group_id: KGid, group_id_present: bool,
  default_permissions: bool, allow_other: bool,
  max_read: u32, blksize: u32,
  subtype: Option<String>,
  is_bdev: bool, destroy: bool,
  legacy_opts_show: bool, no_control: bool, no_force_umount: bool,
  dax_mode: u8, dax_dev: Option<*DaxDevice>,
}
```

`FuseInode::alloc(sb) -> Option<*Inode>`:
1. fi = alloc_inode_sb(sb, fuse_inode_cachep, GFP_KERNEL)?.
2. memset((u8*)fi + sizeof(fi.inode), 0, sizeof(*fi) - sizeof(fi.inode)).
3. fi.inval_mask = !0; mutex_init; spin_lock_init.
4. fi.forget = fuse_alloc_forget()?  /* on fail goto out_free */
5. #[cfg(fuse_dax)] fuse_dax_inode_alloc(sb, fi).
6. #[cfg(fuse_passthrough)] fuse_inode_backing_set(fi, NULL).
7. Some(&fi.inode).

`FuseInode::change_attributes_common(inode, attr, sx, attr_valid, cache_mask, evict_ctr)`:
1. lockdep_assert_held(&fi.lock).
2. cond = !evict_ctr ∨ fi.attr_version != 0 ∨ evict_ctr == fc.evict_ctr.
3. if cond: set_mask_bits(&fi.inval_mask, STATX_BASIC_STATS, 0).
4. fi.attr_version = atomic64_inc_return(&fc.attr_version).
5. fi.i_time = attr_valid.
6. i_ino = fuse_squash_ino(attr.ino).
7. i_mode = (i_mode & S_IFMT) | (attr.mode & 0o7777).
8. set_nlink; i_uid; i_gid; i_blocks.
9. Clamp attr.[a/m/c]timensec to NSEC_PER_SEC-1.
10. inode_set_atime; if !(cache_mask & STATX_MTIME) set mtime; same ctime.
11. If sx: clamp btime nsec; clear STATX_BTIME inval; if STATX_BTIME set FUSE_I_BTIME + fi.i_btime.
12. cached_i_blkbits = attr.blksize ? ilog2(attr.blksize) : sb.s_blocksize_bits.
13. fi.orig_i_mode = i_mode; if !fc.default_permissions: i_mode &= !S_ISVTX.
14. fi.orig_ino = attr.ino; i_flags &= !S_NOSEC.

`FuseInode::iget(sb, nodeid, generation, attr, attr_valid, attr_version, evict_ctr) -> Option<*Inode>`:
1. If fc.auto_submounts ∧ FUSE_ATTR_SUBMOUNT ∧ S_ISDIR(attr.mode):
   - inode = new_inode(sb); fuse_init_inode; fi.nodeid = nodeid; submount_lookup allocate + init; i_flags |= S_AUTOMOUNT; goto done.
2. retry: inode = iget5_locked(sb, nodeid, eq, set, &nodeid)?.
3. is_new = I_NEW set.
4. If is_new:
   - i_flags |= S_NOATIME.
   - If !fc.writeback_cache ∨ !S_ISREG(attr.mode): i_flags |= S_NOCMTIME.
   - i_generation = generation; fuse_init_inode.
5. Else if fuse_stale_inode(inode, generation, attr):
   - fuse_make_bad(inode); if !root: remove_inode_hash + iput; goto retry.
6. fi.nlookup += 1.
7. done: Self::change_attributes_i(inode, attr, None, attr_valid, attr_version, evict_ctr).
8. If is_new: unlock_new_inode.

`FuseFsCtx::parse_param(fsc, param) -> Result<()>`:
1. If FS_CONTEXT_FOR_RECONFIGURE: oldapi ⟹ return 0 (back-compat); else invalfc.
2. opt = fs_parse(fuse_fs_parameters, param, &result)?.
3. Match opt {
   - OPT_SOURCE: dedup; fsc.source = param.string.
   - OPT_SUBTYPE: dedup; ctx.subtype = param.string.
   - OPT_FD: per-fs_value_is_file: fuse_opt_fd(file); else fget(uint_32) + fuse_opt_fd.
   - OPT_ROOTMODE: fuse_valid_type check; ctx.rootmode + rootmode_present.
   - OPT_USER_ID: kuid_has_mapping(fsc.user_ns); ctx.user_id + user_id_present.
   - OPT_GROUP_ID: kgid_has_mapping; ctx.group_id + group_id_present.
   - OPT_DEFAULT_PERMISSIONS: ctx.default_permissions = true.
   - OPT_ALLOW_OTHER: ctx.allow_other = true.
   - OPT_MAX_READ: ctx.max_read = uint_32.
   - OPT_BLKSIZE: if !ctx.is_bdev: invalfc; else ctx.blksize = uint_32.
   - _: -EINVAL. }.

`FuseConn::send_init(fm) -> Result<()>`:
1. ia = Self::new_init(fm).
2. If fm.fc.sync_init:
   - args.abort_on_kill = true.
   - err = fuse_simple_request(fm, &ia.args); if err > 0: err = 0.
3. Else:
   - args.end = process_init_reply.
   - err = fuse_simple_background(fm, &ia.args, GFP_KERNEL); if !err: return Ok.
4. process_init_reply(fm, &ia.args, err).
5. If fm.fc.conn_error: return Err(-ENOTCONN).

`FuseConn::process_init_reply(fm, args, error)`:
1. fc = fm.fc; ia = container_of(args); arg = &ia.out; ok = true.
2. If error ∨ arg.major != FUSE_KERNEL_VERSION: ok = false.
3. Else:
   - process_init_limits(fc, arg).
   - If arg.minor >= 6: import flag matrix per REQ-25 onto fc.*; bound max_pages; switch fid_operations on FUSE_NO_EXPORT_SUPPORT; SB_NOSEC on FUSE_HANDLE_KILLPRIV_V2; mutually exclude PASSTHROUGH ∧ WRITEBACK_CACHE.
   - init_server_timeout(fc, timeout).
   - sb.s_bdi.ra_pages = min(existing, ra_pages).
   - fc.minor = arg.minor; max_write clamp; conn_init = 1.
4. If !ok: conn_init = 0; conn_error = 1.
5. fuse_set_initialized(fc); wake_up_all(&fc.blocked_waitq).

`FuseSb::fill_common(sb, ctx) -> Result<()>`:
1. If SB_MANDLOCK: return Err(-EINVAL).
2. fc.curr_bucket = fuse_sync_bucket_alloc().
3. fuse_sb_defaults(sb).
4. If ctx.is_bdev: sb_set_blocksize(ctx.blksize)?; fc.sync_fs = 1. Else: PAGE_SIZE.
5. s_subtype = ctx.subtype; fuse_dax_conn_alloc (DAX).
6. fc.dev = sb.s_dev; fm.sb = sb.
7. fuse_bdi_init(fc, sb)?.
8. If SB_POSIXACL: fc.dont_mask = 1. sb.s_flags |= SB_POSIXACL.
9. fc.default_permissions / allow_other / user_id / group_id / legacy_opts_show / destroy / no_control / no_force_umount = ctx.*.
10. fc.max_read = max(4096, ctx.max_read).
11. root = fuse_get_root_inode(sb, ctx.rootmode); set_default_d_op(fuse_dentry_operations); root_dentry = d_make_root(root)?.
12. mutex_lock(&fuse_mutex).
13. If fud: if fuse_dev_fc_get already set: -EINVAL; if fud.sync_init: fc.sync_init = 1.
14. fuse_ctl_add_conn(fc)?.
15. list_add_tail(&fc.entry, &fuse_conn_list).
16. sb.s_root = root_dentry.
17. If fud: fuse_dev_install(fud, fc); wake_up_all(&fuse_dev_waitq).
18. mutex_unlock(&fuse_mutex).

`FuseSb::fill(sb, fsc) -> Result<()>`:
1. If !ctx.fud ∨ !rootmode_present ∨ !user_id_present ∨ !group_id_present: -EINVAL.
2. FuseSb::fill_common(sb, ctx)?.
3. FuseConn::send_init(fm).

`FuseFs::kill_sb_anon(sb)`:
1. FuseSb::destroy(sb).
2. kill_anon_super(sb).
3. FuseMount::destroy(fm).

`FuseSb::destroy(sb)`:
1. fm = get_fuse_mount_super(sb).
2. if sb.s_root:
   - last = FuseMount::remove(fm).
   - if last: FuseConn::destroy(fm) /* FUSE_DESTROY + abort + wait + ctl_remove + list_del */.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fuse_inode_offset_zero` | INVARIANT | per-FuseInode: offsetof(inode) == 0 (BUILD_BUG_ON). |
| `alloc_inval_mask_all_set` | INVARIANT | per-alloc_inode: inval_mask = !0 on return. |
| `evict_queues_forget_when_active` | INVARIANT | per-evict_inode: SB_ACTIVE ∧ nlookup>0 ⟹ fuse_queue_forget called. |
| `iget_stale_makes_bad` | INVARIANT | per-iget: stale generation ⟹ fuse_make_bad + retry. |
| `parse_param_fd_user_ns_match` | INVARIANT | per-fuse_opt_fd: file.f_cred.user_ns != fsc.user_ns ⟹ invalfc. |
| `parse_param_fd_must_be_fuse_dev` | INVARIANT | per-fuse_opt_fd: file.f_op != fuse_dev_operations ⟹ invalfc. |
| `fill_super_required_opts` | INVARIANT | per-fuse_fill_super: missing fud / rootmode / user_id / group_id ⟹ -EINVAL. |
| `init_reply_rejects_wrong_major` | INVARIANT | per-process_init_reply: arg.major != FUSE_KERNEL_VERSION ⟹ conn_error. |
| `posix_acl_implies_default_permissions` | INVARIANT | per-init_reply: FUSE_POSIX_ACL ⟹ default_permissions=1 ∧ posix_acl=1. |
| `passthrough_excludes_writeback_cache` | INVARIANT | per-init_reply: FUSE_PASSTHROUGH active ⟹ !(flags & FUSE_WRITEBACK_CACHE). |
| `non_cap_clamps_bg_limits` | INVARIANT | per-process_init_limits: !cap_sys_admin ⟹ clamp ≤ max_user_bgreq / max_user_congthresh. |
| `sb_destroy_last_mount_destroys_conn` | INVARIANT | per-sb_destroy: list_empty(fc.mounts) ⟹ fuse_conn_destroy. |
| `umount_no_force_skips_abort` | INVARIANT | per-umount_begin: fc.no_force_umount ⟹ early-return. |

### Layer 2: TLA+

`fs/fuse/inode.tla`:
- Per-iget + per-change_attributes + per-FUSE_INIT handshake + per-evict-forget + per-sb-teardown + per-bucket-rollover.
- Properties:
  - `safety_offsetof_inode_zero` — per-FuseInode layout invariant.
  - `safety_init_reply_caps_consistent` — per-INIT: applied caps are consistent (no passthrough+writeback_cache, etc.).
  - `safety_evict_forget_for_active_inode` — per-evict: SB_ACTIVE ⟹ FORGET enqueued for nlookup.
  - `safety_attr_cache_ttl_monotone` — per-attr_version: monotonically increasing under fc.attr_version.
  - `safety_sb_teardown_destroys_conn_once` — per-sb-destroy: FUSE_DESTROY sent at most once per fc.
  - `liveness_per_INIT_resolves_blocked_waitq` — per-INIT: eventually wake_up_all(&blocked_waitq).
  - `liveness_per_evict_forget_eventually_delivered` — per-evict: FORGET enqueued ⟹ eventually delivered (or fc aborted).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `FuseInode::alloc` post: inval_mask == !0 ∧ forget allocated | `FuseInode::alloc` |
| `FuseInode::free` post: forget freed, dax freed, backing put, cache freed | `FuseInode::free` |
| `FuseInode::evict` post: SB_ACTIVE ∧ nlookup>0 ⟹ forget enqueued | `FuseInode::evict` |
| `FuseInode::change_attributes` post: fi.attr_version strictly increasing | `FuseInode::change_attributes` |
| `FuseInode::iget` post: existing ⟹ nlookup += 1; stale ⟹ make_bad+retry | `FuseInode::iget` |
| `FuseFsCtx::parse_param` post: invalid fd ⟹ Err; option encoded into ctx | `FuseFsCtx::parse_param` |
| `FuseSb::fill_common` post: sb.s_root set ∧ fc on fuse_conn_list | `FuseSb::fill_common` |
| `FuseSb::fill` post: ok ⟹ FUSE_INIT scheduled or completed | `FuseSb::fill` |
| `FuseConn::process_init_reply` post: ok ⟹ fc.conn_init=1; !ok ⟹ fc.conn_error=1 | `FuseConn::process_init_reply` |
| `FuseSb::destroy` post: last mount ⟹ fuse_conn_destroy invoked once | `FuseSb::destroy` |

### Layer 4: Verus/Creusot functional

`Per-mount(/dev/fuse fd, options) → fuse_init_fs_context → fuse_parse_param × N → fuse_get_tree → fuse_fill_super → fuse_fill_super_common → fuse_send_init → process_init_reply (negotiate caps) → SB_ACTIVE` and `per-umount: fuse_kill_sb_* → fuse_sb_destroy → fuse_mount_remove → fuse_conn_destroy → fuse_mount_destroy → fuse_conn_put → delayed_release` semantic equivalence: per-Documentation/filesystems/fuse.rst + include/uapi/linux/fuse.h FUSE_KERNEL_VERSION protocol.

## Hardening

(Inherits row-1 features from `fs/fuse/00-overview.md` § Hardening.)

FUSE inode + super reinforcement:

- **Per-fd= must reference /dev/fuse** — defense against per-arbitrary-fd hijack of mount path.
- **Per-fd= same user_ns as fs_context** — defense against per-cross-ns fud takeover.
- **Per-rootmode/user_id/group_id required** — defense against per-uninitialized-credentials mount.
- **Per-FUSE_INIT major-mismatch ⟹ conn_error** — defense against per-protocol-skew memory corruption.
- **Per-non-CAP_SYS_ADMIN bg/congestion clamp** — defense against per-unprivileged-bg-exhaustion.
- **Per-passthrough excludes writeback_cache** — defense against per-double-buffering corruption.
- **Per-passthrough max_stack_depth ≤ FILESYSTEM_MAX_STACK_DEPTH** — defense against per-fs-stacking recursion.
- **Per-evict_ctr increment on nlink>0 evict** — defense against per-evict-vs-LOOKUP-race stale inode.
- **Per-FUSE_ATTR_SUBMOUNT not inserted into inode hash** — defense against per-nodeid-collision in submount.
- **Per-fuse_make_bad on stale generation** — defense against per-nodeid-reuse stale I/O.
- **Per-S_NOATIME default + S_NOCMTIME unless writeback_cache** — defense against per-VFS-driven attr churn.
- **Per-default_permissions enforces S_ISVTX preservation** — defense against per-may_delete sticky-bit failure.
- **Per-SB_I_NOIDMAP unless FUSE_ALLOW_IDMAP ∧ default_permissions** — defense against per-idmap-without-server-support.
- **Per-SB_I_UNTRUSTED_MOUNTER for non-init-userns** — defense against per-unprivileged-mount LSM bypass.
- **Per-bdi_set_max_ratio(1)** — defense against per-FUSE-driver dirty-page DoS.
- **Per-curr_bucket RCU rollover** — defense against per-syncfs lost-write window.
- **Per-FUSE_HANDLE_KILLPRIV_V2 ⟹ SB_NOSEC** — defense against per-stale security.capability after server xattr changes.
- **Per-S_NOSEC cleared on attr refresh** — defense against per-server-side suid/sgid/security.capability tampering.
- **Per-FUSE_DESTROY before kill_sb** — defense against per-userspace-server orphan state.
- **Per-killsb rwsem on mounts list** — defense against per-mount-list mutation race.
- **Per-fuse_abort_conn on umount_begin** — defense against per-hanging-mount on userspace-server crash.

## Open Questions

- DAX-mode interaction with FUSE_PASSTHROUGH: are they mutually exclusive at INIT level? (Currently the kernel allows neither flag to gate the other.)
- FUSE_ALLOW_IDMAP without FUSE_POSIX_ACL but with default_permissions: idmap honored, but ACL not enforced — is this exposed as a server-error in init_reply?
- fuse_evict_inode WARN on iocachectr/write_files/queued_writes: covers regular files only; passthrough-backed file lifecycle interaction with these counters needs an explicit assertion.
- FUSE_OVER_IO_URING capability set unilaterally based on global fuse_uring_enabled(): need an explicit per-conn negotiation guard.

## Out of Scope

- fs/fuse/dev.c /dev/fuse character device (covered in `dev.md` Tier-3)
- fs/fuse/file.c regular-file ops (covered in `file.md` Tier-3 when expanded)
- fs/fuse/dir.c directory ops (covered in `dir.md` Tier-3 when expanded)
- fs/fuse/control.c sysfs control plane (covered in `control.md` Tier-3 when expanded)
- fs/fuse/dax.c DAX integration (covered separately if expanded)
- fs/fuse/passthrough.c passthrough/backing-file plumbing (covered separately if expanded)
- fs/fuse/dev_uring.c io_uring transport (covered separately if expanded)
- fs/fuse/virtio_fs.c virtiofs transport (covered separately if expanded)
- fs/fuse/xattr.c xattr handlers (covered separately if expanded)
- fs/fuse/acl.c POSIX ACL (covered separately if expanded)
- Implementation code
