---
title: "Tier-3: fs/9p/vfs_super.c — 9P client filesystem VFS superblock"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **9P virtual filesystem (v9fs)** mounts a remote hierarchy exported by a server speaking the Plan-9 9P wire-protocol (variants `9p2000`, `9p2000.u`, `9p2000.L`). `fs/9p/vfs_super.c` is the VFS face: per-mount it allocates a `struct v9fs_session_info`, creates a `struct p9_client` over the requested transport (`fd`, `tcp`, `unix`, `virtio`, `rdma`, `xen`), issues a 9P `Tattach` to obtain the root `struct p9_fid`, fills the `struct super_block` with `V9FS_MAGIC` and per-protocol `super_operations` (`v9fs_super_ops` vs `v9fs_super_ops_dotl`), and attaches the root fid to the root dentry. Per-`v9fs_get_tree`: builds `sb`. Per-`v9fs_kill_super`: cancels in-flight 9P requests, destroys the client, frees the session. Per-`v9fs_show_options`: re-renders the mount string in `/proc/mounts`. Per-`v9fs_statfs`: issues `Tstatfs` (dotl) or falls back to `simple_statfs`. Path walks (covered in `vfs/walk.md` Tier-3 mapping) are NFS-like: each component is resolved via a 9P `Twalk` from the parent fid, with `Tclunk` releasing fids on drop. Critical for: container-image distribution (virtio-9p), VM guest-shared filesystems (QEMU `-virtfs`), diskless boot, Plan-9 interoperability.

This Tier-3 covers `fs/9p/vfs_super.c` (~365 lines).

### Acceptance Criteria

- [ ] AC-1: `mount -t 9p -o trans=virtio,version=9p2000.L share /mnt/9p`: succeeds; root inode reflects server attach.
- [ ] AC-2: `v9fs_fill_super`: sb.s_magic == V9FS_MAGIC; sb.s_op == v9fs_super_ops_dotl for `.L`; v9fs_super_ops for legacy.
- [ ] AC-3: `v9fs_session_init`: failure of `p9_client_create` returns the propagated err (-ECONNREFUSED, -ETIMEDOUT, -EINVAL) and frees uname/aname.
- [ ] AC-4: `v9fs_session_init`: failure of `p9_client_attach` (Tattach denied) destroys client and returns -EACCES / -EPERM.
- [ ] AC-5: `cat /proc/mounts | grep 9p` after mount: re-renders all non-default options including trans/version/msize (via `p9_show_client_options`).
- [ ] AC-6: `umount /mnt/9p`: v9fs_kill_super invokes p9_client_destroy; no fid leaks; session removed from v9fs_sessionlist.
- [ ] AC-7: `umount -f /mnt/9p` on a stuck transport: v9fs_umount_begin → begin_cancel → only Tclunk RPCs accepted; umount progresses.
- [ ] AC-8: `statvfs` on `.L` mount: returns server-reported {bsize, blocks, bfree, files} via Tstatfs.
- [ ] AC-9: `statvfs` on legacy `9p2000.u` mount: falls back to simple_statfs (synthetic 0/0 values).
- [ ] AC-10: With `cache=none`: every stat() hits the wire (drop_inode returns 1); BDI readahead = 0.
- [ ] AC-11: With `cache=loose`: dentry ops = v9fs_cached_dentry_operations; BDI readahead = maxdata >> PAGE_SHIFT.
- [ ] AC-12: `v9fs_get_tree` failure path after `sget_fc`: deactivate_locked_super(sb) + p9_fid_put(fid); no leak.
- [ ] AC-13: Deep-path lookup (>16 components): chained `Twalk`s succeed; intermediate fids `Tclunk`d on error.
- [ ] AC-14: Root fid is `Tclunk`d once on superblock teardown (via dentry release, not double-clunked).

### Architecture

```
struct V9fsSessionInfo {
  flags: u32,
  nodev: u8,
  debug: u16,
  afid: u32,
  cache: u32,
  cachetag: Option<KString>,
  fscache: Option<*FscacheVolume>,
  uname: KString,
  aname: KString,
  maxdata: u32,
  dfltuid: Kuid,
  dfltgid: Kgid,
  uid: Kuid,
  clnt: *P9Client,
  slist: ListHead,
  rename_sem: RwSemaphore,
  session_lock_timeout: i64,
}

struct V9fsContext {                  // fs_context::fs_private
  session_opts: SessionOpts,
  client_opts: P9ClientOpts,
  fd_opts: P9FdOpts,
  rdma_opts: P9RdmaOpts,
}
```

`Vfs9p::get_tree(fc) -> Result<()>`:
1. v9ses = KBox::try_new_zeroed::<V9fsSessionInfo>()?.
2. fid = V9fsSession::init(&mut v9ses, fc)?;       /* p9 attach */
3. fc.s_fs_info = Some(v9ses).
4. sb = sget_fc(fc, None, set_anon_super_fc).inspect_err(|_| clunk(fid))?.
5. Vfs9p::fill_super(sb).inspect_err(|_| {p9_fid_put(fid); deactivate_locked_super(sb)})?.
6. /* per-cache dentry ops */
7. if v9ses.cache & (CACHE_META|CACHE_LOOSE):
   - set_default_d_op(sb, &V9FS_CACHED_DENTRY_OPS).
8. else:
   - set_default_d_op(sb, &V9FS_DENTRY_OPS).
   - sb.s_d_flags |= DCACHE_DONTCACHE.
9. inode = V9fsInode::get_new_from_fid(v9ses, fid, sb)?.
10. root = d_make_root(inode).ok_or(-ENOMEM)?.
11. sb.s_root = root.
12. V9fsAcl::get(inode, fid)?.
13. V9fsFid::add(root, &mut Some(fid)).             /* root fid → root dentry */
14. fc.root = dget(sb.s_root).
15. Ok(()).

`Vfs9p::fill_super(sb) -> Result<()>`:
1. v9ses = sb.s_fs_info.
2. sb.s_maxbytes = MAX_LFS_FILESIZE.
3. sb.s_blocksize_bits = fls(v9ses.maxdata - 1).
4. sb.s_blocksize = 1 << s_blocksize_bits.
5. sb.s_magic = V9FS_MAGIC.
6. if v9fs_proto_dotl(v9ses):
   - sb.s_op = &V9FS_SUPER_OPS_DOTL.
   - if !(v9ses.flags & V9FS_NO_XATTR): sb.s_xattr = &V9FS_XATTR_HANDLERS.
7. else:
   - sb.s_op = &V9FS_SUPER_OPS.
   - sb.s_time_max = u32::MAX.
8. sb.s_time_min = 0.
9. super_setup_bdi(sb)?.
10. if v9ses.cache == 0: sb.s_bdi.ra_pages = 0; sb.s_bdi.io_pages = 0.
11. else: sb.s_bdi.ra_pages = sb.s_bdi.io_pages = v9ses.maxdata >> PAGE_SHIFT.
12. sb.s_flags |= SB_ACTIVE.
13. if (v9ses.flags & V9FS_ACL_MASK) == V9FS_POSIX_ACL: sb.s_flags |= SB_POSIXACL.
14. Ok(()).

`V9fsSession::init(v9ses, fc) -> Result<P9Fid>`:
1. init_rwsem(&v9ses.rename_sem).
2. v9ses.clnt = p9_client_create(fc)?;              /* per-trans_mod: fd/tcp/virtio/rdma/xen */
3. /* per-protocol */
4. v9ses.flags = V9FS_ACCESS_USER.
5. if p9_is_proto_dotl(clnt): v9ses.flags = V9FS_ACCESS_CLIENT | V9FS_PROTO_2000L.
6. else if p9_is_proto_dotu(clnt): v9ses.flags |= V9FS_PROTO_2000U.
7. v9fs_apply_options(v9ses, fc).
8. v9ses.maxdata = clnt.msize - P9_IOHDRSZ.
9. /* per-fallback ACCESS adjustments */
10. fid = p9_client_attach(clnt, None, uname, INVALID_UID, aname)?;  /* 9P Tattach */
11. fid.uid = if ACCESS==SINGLE { v9ses.uid } else { INVALID_UID }.
12. if v9ses.cache & CACHE_FSCACHE: v9fs_cache_session_get_cookie(v9ses, fc.source)?.
13. spin_lock(&V9FS_SESSIONLIST_LOCK); v9ses.slist.add(&V9FS_SESSIONLIST); spin_unlock.
14. Ok(fid).

`Vfs9p::kill_super(sb)`:
1. v9ses = sb.s_fs_info.take().
2. kill_anon_super(sb).
3. V9fsSession::cancel(&v9ses).
4. V9fsSession::close(v9ses).         /* p9_client_destroy + free */

`V9fsSession::close(v9ses)`:
1. if let Some(clnt) = v9ses.clnt.take(): p9_client_destroy(clnt).
2. fscache_relinquish_volume(v9fs_session_cache(v9ses), None, false).
3. drop(v9ses.cachetag).
4. drop(v9ses.uname); drop(v9ses.aname).
5. spin_lock(&V9FS_SESSIONLIST_LOCK); v9ses.slist.del(); spin_unlock.

`Vfs9p::statfs(dentry, buf) -> Result<()>`:
1. fid = V9fsFid::lookup(dentry)?;
2. v9ses = v9fs_dentry2v9ses(dentry).
3. if v9fs_proto_dotl(v9ses):
   - res = p9_client_statfs(fid, &mut rs).
   - if res == 0: buf = kstatfs::from_p9_rstatfs(rs); p9_fid_put(fid); return Ok(()).
   - if res != -ENOSYS: p9_fid_put(fid); return Err(res).
4. /* legacy fallback */
5. let r = simple_statfs(dentry, buf).
6. p9_fid_put(fid).
7. r.

`Vfs9p::show_options(m, root) -> Result<()>`:
- Render every non-default field of v9ses as `,name[=val]` entries via seq_printf / seq_puts.
- Then chain to p9_show_client_options for transport-level keys (`trans=`, `msize=`, `version=`, `port=`, …).

`V9fsFid::lookup(dentry) -> Result<*P9Fid>`:
1. Find cached fid on `dentry.d_inode` for `current_uid()` (or `INVALID_UID` for ANY).
2. Miss → walk up `d_parent` chain until a cached fid found.
3. Collect intermediate path components (most-recent first), reverse.
4. Per-chunk of ≤ `P9_MAXWELEM` (16) components: issue `p9_client_walk(parent_fid, newfid, wnames)`.
5. On any walk error: `p9_client_clunk(intermediate_fid)`; return Err.
6. Final newfid bound to dentry via V9fsFid::add; return ref-incremented pointer.

### Out of Scope

- `net/9p/client.c` (9P RPC engine — covered separately in `net/9p/client.md` Tier-3)
- `net/9p/trans_{fd,virtio,rdma,xen}.c` (transport implementations — separate Tier-3s)
- `fs/9p/vfs_inode.c` / `vfs_inode_dotl.c` (inode-ops; covered in `fs/9p/vfs-inode.md` Tier-3 if expanded)
- `fs/9p/vfs_file.c` (file-ops; separate)
- `fs/9p/vfs_addr.c` (netfs aops; separate)
- `fs/9p/vfs_dir.c` (readdir; separate)
- `fs/9p/cache.c` (fscache integration; covered in `fs/fscache.md` if expanded)
- `fs/9p/xattr.c` / `acl.c` (xattr + ACL; separate)
- `fs/9p/fid.c` walk path (covered above only in REQ-16 / Architecture; full fid table in `fs/9p/fid.md` if expanded)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct v9fs_session_info` | per-mount session state | `V9fsSessionInfo` |
| `struct v9fs_context` | per-mount fs_context private | `V9fsContext` |
| `v9fs_fs_type` | per-`struct file_system_type` "9p" | `Vfs9p::FS_TYPE` |
| `v9fs_init_fs_context()` | per-`init_fs_context` callback | `Vfs9p::init_fs_context` |
| `v9fs_context_ops` | per-`fs_context_operations` table | `Vfs9p::CONTEXT_OPS` |
| `v9fs_parse_param()` | per-mount-option parse | `Vfs9p::parse_param` |
| `v9fs_param_spec` | per-parameter spec table | `Vfs9p::PARAM_SPEC` |
| `v9fs_get_tree()` | per-`get_tree` build superblock | `Vfs9p::get_tree` |
| `v9fs_fill_super()` | per-`fill_super` init sb fields | `Vfs9p::fill_super` |
| `v9fs_kill_super()` | per-`kill_sb` teardown | `Vfs9p::kill_super` |
| `v9fs_umount_begin()` | per-umount2 MNT_FORCE hook | `Vfs9p::umount_begin` |
| `v9fs_free_fc()` | per-`fs_context` free | `Vfs9p::free_fc` |
| `v9fs_session_init()` | per-`p9_client_create` + `Tattach` | `V9fsSession::init` |
| `v9fs_session_close()` | per-`p9_client_destroy` + free | `V9fsSession::close` |
| `v9fs_session_cancel()` | per-`p9_client_disconnect` (transport) | `V9fsSession::cancel` |
| `v9fs_session_begin_cancel()` | per-`p9_client_begin_disconnect` | `V9fsSession::begin_cancel` |
| `v9fs_show_options()` | per-`/proc/mounts` render | `V9fsSession::show_options` |
| `v9fs_statfs()` | per-`statfs` `Tstatfs` dispatch | `Vfs9p::statfs` |
| `v9fs_drop_inode()` | per-`drop_inode` cache policy | `Vfs9p::drop_inode` |
| `v9fs_write_inode()` / `_dotl()` | per-`writeback_control` flush | `Vfs9p::write_inode` |
| `v9fs_super_ops` / `v9fs_super_ops_dotl` | per-protocol vtable | shared |
| `v9fs_fid_add()` | per-attach fid to dentry | `V9fsFid::add` |
| `v9fs_fid_lookup()` | per-walk-or-find fid for dentry | `V9fsFid::lookup` |
| `V9FS_MAGIC` | per-`sb->s_magic` | constant |

### compatibility contract

REQ-1: struct v9fs_session_info:
- flags: per-`V9FS_ACCESS_{USER,ANY,CLIENT,SINGLE}`, `V9FS_PROTO_2000{U,L}`, `V9FS_NO_XATTR`, `V9FS_POSIX_ACL`, `V9FS_DIRECT_IO`, `V9FS_IGNORE_QV`.
- nodev: per-`nodevmap` option.
- debug: per-debug-level mask.
- afid: per-authentication-fid (`~0` = none).
- cache: per-`CACHE_{NONE,META,LOOSE,FSCACHE,READAHEAD,MMAP}`.
- cachetag: per-fscache volume tag (CONFIG_9P_FSCACHE).
- fscache: per-`struct fscache_volume` cookie.
- uname: per-9P user-name (default `V9FS_DEFUSER` = "nobody").
- aname: per-9P attach-name (mount-specifier, default `V9FS_DEFANAME`).
- maxdata: per-`p9_client.msize - P9_IOHDRSZ` (max payload per message).
- dfltuid / dfltgid: per-legacy uid/gid mapping.
- uid: per-`V9FS_ACCESS_SINGLE` numeric-uid owner.
- clnt: pointer to `struct p9_client` instantiated for this mount.
- slist: per-`v9fs_sessionlist` link.
- rename_sem: per-rename-coherency rw_semaphore.
- session_lock_timeout: per-blocking-9P-lock retry interval (default `P9_LOCK_TIMEOUT`).

REQ-2: struct v9fs_context (fs_context private):
- session_opts: per-`v9fs_session_info` field accumulator (uname/aname/cachetag/uid/...).
- client_opts: per-`p9_client_opts` (proto_version, msize, trans_mod).
- fd_opts: per-`fd` transport (port, rfd, wfd, privport).
- rdma_opts: per-`rdma` transport (port, sq_depth, rq_depth, timeout, privport).

REQ-3: v9fs_init_fs_context(fc):
- ctx = kzalloc(v9fs_context).
- if !ctx: return -ENOMEM.
- fc.ops = &v9fs_context_ops.
- fc.fs_private = ctx.
- /* defaults */
- ctx.session_opts.afid = ~0.
- ctx.session_opts.cache = CACHE_NONE.
- ctx.session_opts.session_lock_timeout = P9_LOCK_TIMEOUT.
- ctx.session_opts.uname = kstrdup(V9FS_DEFUSER).
- ctx.session_opts.aname = kstrdup(V9FS_DEFANAME).
- ctx.session_opts.uid = INVALID_UID.
- ctx.session_opts.dfltuid = V9FS_DEFUID.
- ctx.session_opts.dfltgid = V9FS_DEFGID.
- ctx.client_opts.proto_version = p9_proto_2000L (default `.L`).
- ctx.client_opts.msize = DEFAULT_MSIZE.
- ctx.fd_opts.port = P9_FD_PORT; rfd = wfd = ~0; privport = false.
- ctx.rdma_opts.port = P9_RDMA_PORT; sq_depth = P9_RDMA_SQ_DEPTH; rq_depth = P9_RDMA_RQ_DEPTH; timeout = P9_RDMA_TIMEOUT; privport = false.
- return 0.

REQ-4: v9fs_get_tree(fc):
- /* Per-allocation */
- v9ses = kzalloc(v9fs_session_info).
- if !v9ses: return -ENOMEM.
- /* Per-9P attach */
- fid = v9fs_session_init(v9ses, fc).
- if IS_ERR(fid): goto free_session with err.
- fc.s_fs_info = v9ses.
- /* Per-anonymous-superblock */
- sb = sget_fc(fc, NULL, set_anon_super_fc).
- if IS_ERR(sb): goto clunk_fid.
- /* Fill */
- retval = v9fs_fill_super(sb).
- if retval: goto release_sb.
- /* Per-cache dentry-ops */
- if v9ses.cache & (CACHE_META | CACHE_LOOSE):
  - set_default_d_op(sb, &v9fs_cached_dentry_operations).
- else:
  - set_default_d_op(sb, &v9fs_dentry_operations).
  - sb.s_d_flags |= DCACHE_DONTCACHE.
- /* Root inode + dentry */
- inode = v9fs_get_new_inode_from_fid(v9ses, fid, sb).
- root = d_make_root(inode).
- if !root: retval = -ENOMEM; goto release_sb.
- sb.s_root = root.
- retval = v9fs_get_acl(inode, fid).
- v9fs_fid_add(root, &fid).        /* root fid handed off to dentry */
- fc.root = dget(sb.s_root).
- return 0.

REQ-5: v9fs_fill_super(sb):
- v9ses = sb.s_fs_info.
- sb.s_maxbytes = MAX_LFS_FILESIZE.
- sb.s_blocksize_bits = fls(v9ses.maxdata - 1).
- sb.s_blocksize = 1 << s_blocksize_bits.
- sb.s_magic = V9FS_MAGIC.
- /* Per-protocol vtable */
- if v9fs_proto_dotl(v9ses):
  - sb.s_op = &v9fs_super_ops_dotl.
  - if !(v9ses.flags & V9FS_NO_XATTR): sb.s_xattr = v9fs_xattr_handlers.
- else:
  - sb.s_op = &v9fs_super_ops.
  - sb.s_time_max = U32_MAX (legacy 32-bit time wall).
- sb.s_time_min = 0.
- ret = super_setup_bdi(sb).
- if ret: return ret.
- /* Per-cache BDI readahead window */
- if !v9ses.cache: sb.s_bdi.ra_pages = sb.s_bdi.io_pages = 0.
- else: sb.s_bdi.ra_pages = sb.s_bdi.io_pages = v9ses.maxdata >> PAGE_SHIFT.
- sb.s_flags |= SB_ACTIVE.
- /* Per-CONFIG_9P_FS_POSIX_ACL */
- if (v9ses.flags & V9FS_ACL_MASK) == V9FS_POSIX_ACL: sb.s_flags |= SB_POSIXACL.
- return 0.

REQ-6: v9fs_session_init(v9ses, fc):
- init_rwsem(&v9ses.rename_sem).
- /* Per-transport client */
- v9ses.clnt = p9_client_create(fc).
- if IS_ERR(clnt): goto err_names.
- /* Per-protocol flags */
- v9ses.flags = V9FS_ACCESS_USER.
- if p9_is_proto_dotl(clnt): v9ses.flags = V9FS_ACCESS_CLIENT | V9FS_PROTO_2000L.
- else if p9_is_proto_dotu(clnt): v9ses.flags |= V9FS_PROTO_2000U.
- v9fs_apply_options(v9ses, fc).      /* copy parsed opts into v9ses */
- v9ses.maxdata = clnt.msize - P9_IOHDRSZ.
- /* Per-legacy access-mode fallbacks */
- if !dotl ∧ ACCESS == ACCESS_CLIENT: ACCESS = USER.
- if !dotu ∧ !dotl ∧ ACCESS == USER: ACCESS = ANY; uid = INVALID_UID.
- if !dotl ∨ ACCESS != ACCESS_CLIENT: flags &= ~V9FS_ACL_MASK.
- /* Per-9P Tattach */
- fid = p9_client_attach(clnt, NULL, uname, INVALID_UID, aname).
- if IS_ERR(fid): goto err_clnt.
- fid.uid = (ACCESS == SINGLE) ? v9ses.uid : INVALID_UID.
- /* Per-fscache cookie */
- if v9ses.cache & CACHE_FSCACHE: v9fs_cache_session_get_cookie(v9ses, fc.source).
- /* Per-global session list */
- spin_lock(&v9fs_sessionlist_lock); list_add(&v9ses.slist, &v9fs_sessionlist); spin_unlock.
- return fid.

REQ-7: v9fs_kill_super(s):
- v9ses = s.s_fs_info.
- kill_anon_super(s).                /* generic VFS sb teardown */
- v9fs_session_cancel(v9ses).        /* p9_client_disconnect: kill in-flight */
- v9fs_session_close(v9ses).         /* p9_client_destroy + free strings */
- kfree(v9ses); s.s_fs_info = NULL.

REQ-8: v9fs_session_close(v9ses):
- if v9ses.clnt: p9_client_destroy(v9ses.clnt); v9ses.clnt = NULL.
- /* fscache */
- fscache_relinquish_volume(v9fs_session_cache(v9ses), NULL, false).
- kfree(v9ses.cachetag).
- kfree(v9ses.uname); kfree(v9ses.aname).
- spin_lock(&v9fs_sessionlist_lock); list_del(&v9ses.slist); spin_unlock.

REQ-9: v9fs_session_cancel / _begin_cancel:
- cancel: p9_client_disconnect(clnt) — mark transport disconnected; flush pending.
- begin_cancel: p9_client_begin_disconnect(clnt) — refuse all new requests except Tclunk.
- begin_cancel is called from umount_begin (MNT_FORCE).

REQ-10: v9fs_show_options(m, root):
- v9ses = root.d_sb.s_fs_info.
- if debug: seq_printf(",debug=%#x", debug).
- if dfltuid != V9FS_DEFUID: seq_printf(",dfltuid=%u", ...).
- if dfltgid != V9FS_DEFGID: seq_printf(",dfltgid=%u", ...).
- if afid != ~0: seq_printf(",afid=%u").
- if uname != V9FS_DEFUSER: seq_printf(",uname=%s").
- if aname != V9FS_DEFANAME: seq_printf(",aname=%s").
- if nodev: ",nodevmap".
- if cache: ",cache=%#x".
- if cachetag ∧ (cache & CACHE_FSCACHE): ",cachetag=%s".
- access switch: USER | ANY | CLIENT | SINGLE.
- if V9FS_IGNORE_QV: ",ignoreqv".
- if V9FS_DIRECT_IO: ",directio".
- if V9FS_POSIX_ACL: ",posixacl".
- if V9FS_NO_XATTR: ",noxattr".
- return p9_show_client_options(m, v9ses.clnt). /* trans/msize/version/... */

REQ-11: v9fs_statfs(dentry, buf):
- fid = v9fs_fid_lookup(dentry).
- if IS_ERR(fid): return PTR_ERR.
- v9ses = v9fs_dentry2v9ses(dentry).
- if v9fs_proto_dotl(v9ses):
  - res = p9_client_statfs(fid, &rs).
  - if res == 0: buf gets {type, bsize, blocks, bfree, bavail, files, ffree, fsid, namelen}.
  - if res != -ENOSYS: goto done.
- /* Fallback for legacy 9p2000 / 9p2000.u */
- res = simple_statfs(dentry, buf).
- p9_fid_put(fid).
- return res.

REQ-12: v9fs_drop_inode(inode):
- if cache & (CACHE_META | CACHE_LOOSE): return inode_generic_drop(inode).
- /* Non-cached: always drop so next stat hits the wire */
- return 1.

REQ-13: v9fs_write_inode / _dotl:
- Send fsync irrespective of `wbc->sync_mode` via netfs_unpin_writeback.

REQ-14: v9fs_super_ops vtable (legacy 9p2000 / 9p2000.u):
- .alloc_inode = v9fs_alloc_inode.
- .free_inode = v9fs_free_inode.
- .statfs = simple_statfs.
- .drop_inode = v9fs_drop_inode.
- .evict_inode = v9fs_evict_inode.
- .show_options = v9fs_show_options.
- .umount_begin = v9fs_umount_begin.
- .write_inode = v9fs_write_inode.

REQ-15: v9fs_super_ops_dotl vtable (9p2000.L):
- Same as REQ-14 except .statfs = v9fs_statfs, .write_inode = v9fs_write_inode_dotl.

REQ-16: NFS-like long-pathwalk (delegated to fs/9p/fid.c):
- Per-`v9fs_fid_lookup(dentry)`:
  - Find an existing fid on this dentry's inode for the calling uid (cache hit).
  - Else walk from an ancestor whose fid is known up to `dentry->d_sb->s_root`.
  - Build path-component array by walking up `d_parent` until a fid is found.
  - Issue 9P `Twalk(parent_fid, newfid, wnames[])` with up to `P9_MAXWELEM` (16) names per RPC; chain multiple Twalk for deeper paths.
  - On error: `Tclunk(newfid)` and propagate -EREMOTEIO / -EIO.
  - Per-`v9fs_fid_add(dentry, &fid)`: install fid on dentry; ownership transferred (caller's pointer cleared).
- Per-fid lifecycle: every alloc'd fid is `Tclunk`'d on drop (either via dentry release or explicit `p9_fid_put`).

REQ-17: file_system_type "9p":
- .name = "9p".
- .kill_sb = v9fs_kill_super.
- .owner = THIS_MODULE.
- .fs_flags = FS_RENAME_DOES_D_MOVE.
- .init_fs_context = v9fs_init_fs_context.
- .parameters = v9fs_param_spec.

REQ-18: v9fs_umount_begin(sb):
- v9ses = sb.s_fs_info.
- v9fs_session_begin_cancel(v9ses).   /* Block all RPCs except Tclunk so umount -f makes progress. */

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `session_init_rollback_balanced` | INVARIANT | per-`V9fsSession::init`: on err every kalloc'd field freed; clnt destroyed if attach failed. |
| `get_tree_clunk_on_err_after_attach` | INVARIANT | per-`Vfs9p::get_tree`: err after attach but before fid handoff ⟹ `p9_fid_put(fid)`. |
| `kill_super_destroys_client_once` | INVARIANT | per-`Vfs9p::kill_super`: `p9_client_destroy` called exactly once. |
| `super_block_magic_set` | INVARIANT | per-`fill_super` success: `sb.s_magic == V9FS_MAGIC`. |
| `super_ops_per_protocol` | INVARIANT | per-`fill_super`: dotl ⟺ s_op == v9fs_super_ops_dotl; else == v9fs_super_ops. |
| `bdi_ra_pages_per_cache` | INVARIANT | per-`fill_super`: `cache==0 ⟹ ra_pages==0`; else `ra_pages == maxdata>>PAGE_SHIFT`. |
| `fid_lookup_no_leak` | INVARIANT | per-`V9fsFid::lookup` walk: every Twalk that errors is matched by a Tclunk. |
| `umount_begin_disallows_non_clunk` | INVARIANT | per-`umount_begin`: post-call, only Tclunk RPCs accepted on this client. |

### Layer 2: TLA+

`fs/9p/vfs-super.tla`:
- Per-mount lifecycle: init_fs_context → parse_param* → get_tree → fill_super → live → (umount_begin?) → kill_super.
- Properties:
  - `safety_no_double_destroy` — per-client: `p9_client_destroy` invoked at most once.
  - `safety_fid_clunk_paired_with_walk` — per-error in walk: corresponding Tclunk eventually issued.
  - `safety_session_list_balanced` — per-mount: list_add and list_del paired under `v9fs_sessionlist_lock`.
  - `safety_force_umount_progress` — per-umount2(MNT_FORCE): begin_cancel ⟹ stuck RPCs return -EIO within bounded steps.
  - `liveness_per_mount_attaches_or_errors` — per-get_tree: terminates with either sb established or all resources released.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vfs9p::get_tree` post: ok ⟹ `sb.s_root != null ∧ sb.s_magic == V9FS_MAGIC` | `Vfs9p::get_tree` |
| `Vfs9p::fill_super` post: `sb.s_blocksize == 1 << fls(maxdata-1)` | `Vfs9p::fill_super` |
| `V9fsSession::init` post: ok ⟹ `v9ses.clnt != null ∧ fid != null` | `V9fsSession::init` |
| `V9fsSession::init` post: err ⟹ `v9ses.clnt == null ∧ uname/aname freed` | `V9fsSession::init` |
| `Vfs9p::kill_super` post: `sb.s_fs_info == null ∧ session removed from list` | `Vfs9p::kill_super` |
| `Vfs9p::show_options` post: re-mounting with the rendered string yields equivalent v9ses | `Vfs9p::show_options` |
| `V9fsFid::lookup` post: returned fid refers to `dentry.d_inode.qid` | `V9fsFid::lookup` |

### Layer 4: Verus/Creusot functional

`Per-mount: init_fs_context → parse_param → get_tree → (p9_client_create → Tversion → Tattach) → fill_super → root-dentry binding`, and `per-umount: umount_begin? → kill_super → kill_anon_super → cancel → p9_client_destroy`. Semantic equivalence to Linux per `Documentation/filesystems/9p.rst` and the 9P2000 / 9P2000.L specs.

### hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

9P-mount reinforcement:

- **Per-fid clunk on walk error** — defense against per-server-side fid-table exhaustion (UAF-on-server).
- **Per-`p9_client_destroy` exactly-once** — defense against per-double-free of transport buffers.
- **Per-`v9fs_session_begin_cancel` only-Tclunk** — defense against per-stuck-server umount-deadlock.
- **Per-`MAX_LFS_FILESIZE` cap** — defense against per-server-claimed 2^63-byte file overflowing kernel offsets.
- **Per-`maxdata = msize - P9_IOHDRSZ` strict** — defense against per-server replying with > msize (buffer-overrun).
- **Per-`fc.source` provenance for fscache cookie** — defense against per-cookie-collision across mounts.
- **Per-`set_anon_super_fc` (anonymous bdev)** — defense against per-bogus-dev_t aliasing other mounts.
- **Per-`V9FS_NO_XATTR` honored** — defense against per-server-injected hostile xattrs.
- **Per-`session_lock_timeout` bounded** — defense against per-server-never-replies-to-Tlock DoS.
- **Per-uname/aname kstrdup'd into v9ses** — defense against per-fs_context-freed-pointer use.
- **Per-`v9fs_super_ops_dotl` separation** — defense against per-Tstatfs sent to a non-`.L` server.
- **Per-`p9_show_client_options` re-rendering covered by /proc/mounts** — defense against per-remount drift.
- **Per-CAP_SYS_ADMIN required for mount** — defense against per-unprivileged-mount of unauthenticated remote FS.

