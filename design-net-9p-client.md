---
title: "Tier-3: net/9p/client.c — 9P client core (tag/fid/rpc/trans_mod)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`client.c` is the **per-`p9_client` core** of the 9P (Plan 9 file-protocol) stack. It owns: tagged request/reply machinery (`p9_req_t` allocated under `idr` `c->reqs` keyed by 16-bit tag), FID allocation (`idr` `c->fids` keyed by 32-bit FID; `P9_NOFID = ~0`), `msize` negotiation against the server (`p9_client_version` → `Tversion`/`Rversion`), per-protocol-version dialect selection (`9P2000` / `9P2000.u` / `9P2000.L`), and per-transport indirection via `struct p9_trans_module *trans_mod` (resolved by name at mount via `v9fs_get_trans_by_name` or `v9fs_get_default_trans`). It exposes the synchronous request primitive `p9_client_rpc` (waits with `io_wait_event_killable` on `req->wq`, supports `Tflush` cancel) and the zero-copy primitive `p9_client_zc_rpc` (uses `iov_iter` for in/out user-data, allocates only `P9_ZC_HDR_SZ`-byte inline buffer). It implements every 9P operation a 9pfs mount needs: `attach`, `walk`, `open`, `create`, `read`, `write`, `clunk`, `remove`, `stat`/`wstat`, `setattr`/`getattr_dotl`, `xattrwalk`/`xattrcreate`, `readdir`, `mknod`/`mkdir`/`symlink`/`link`/`readlink`, `lock_dotl`/`getlock_dotl`, `fsync`, `statfs`, `rename`/`renameat`, `unlinkat`. Error mapping from `Rerror` (plan9 string + optional `.u` errno) → `-errno` via `p9_errstr2errno`. Critical for: `9pfs` virtio file-passthrough (QEMU `-virtfs`), Xen 9pfs, RDMA-attached fileservers.

This Tier-3 covers `net/9p/client.c` (~2209 lines).

### Acceptance Criteria

- [ ] AC-1: `p9_client_create` with `trans=virtio` resolves to `p9_virtio_trans`; with `trans=tcp` to `p9_tcp_trans`; with omitted `trans=` falls back to `v9fs_get_default_trans`.
- [ ] AC-2: `p9_client_version` sets `c.proto_version` to dialect server agreed; rejects `msize < 4096` with `-EREMOTEIO`.
- [ ] AC-3: `p9_client_rpc(.., P9_TVERSION, ..)` uses reserved tag `P9_NOTAG`; all other RPCs allocate tag in `[0, P9_NOTAG)`.
- [ ] AC-4: `p9_client_rpc` returns successfully on `Rmsg` matching expected type; returns `-errno` on `Rerror`/`Rlerror`; returns `-EIO` if `rc.size > rc.capacity`.
- [ ] AC-5: `p9_client_zc_rpc` does not copy payload through `req->rc.sdata` for the user-data portion; only first `P9_ZC_HDR_SZ` bytes are inline.
- [ ] AC-6: Signal during `p9_client_rpc` → `Tflush` is emitted via `p9_client_flush`; if the response arrives anyway, `-ERESTARTSYS` is suppressed.
- [ ] AC-7: `p9_tag_lookup` under RCU never returns a request with mismatched tag (slab-typesafe re-allocation race).
- [ ] AC-8: `p9_client_disconnect` causes subsequent `p9_client_prepare_req` to return `-EIO`; `begin_disconnect` allows only `P9_TCLUNK`.
- [ ] AC-9: `p9_client_attach(.., NULL, "root", uid, "")` returns a FID with `qid.type & P9_QTDIR` set (server root).
- [ ] AC-10: `p9_client_walk(.., clone=1)` clones FID before walk; on partial walk returns `-ENOENT` and clunks the new FID; original `oldfid` untouched.
- [ ] AC-11: `p9_client_destroy` warns on unclunked FIDs and reclaims `clnt.fids` and `clnt.reqs` idrs; no slab leak.
- [ ] AC-12: `p9_check_errors` for `.L` reads only `d ecode` (single int); for `.u` reads `s ename + d ecode`; for legacy reads only `s ename`.
- [ ] AC-13: `clnt.msize` post-init: `min(user_msize, trans_mod.maxsize, server_msize)`, never less than 4096.
- [ ] AC-14: `safe_errno` rejects out-of-range values with `-EPROTO`.
- [ ] AC-15: `req->refcount` reaches zero exactly once across writer-thread + recv-callback + main-thread; no double-free of `p9_req_t`.

### Architecture

```
struct P9Client {
  lock:          SpinLock,
  msize:         u32,
  proto_version: P9ProtoVersion,    // Legacy | _2000u | _2000L
  status:        AtomicP9Status,    // Connected | BeginDisconnect | Disconnected
  trans_mod:     Option<&'static P9TransModule>,
  trans:         *mut c_void,       // transport-private
  fids:          Idr<P9Fid>,        // [0, P9_NOFID-1]
  reqs:          Idr<P9Req>,        // [0, P9_NOTAG)  + P9_NOTAG slot for Tversion
  fcall_cache:   *mut KmemCache,
  name:          [u8; HOST_NAME_MAX + 1],
}

struct P9Req {
  tc:       P9Fcall,                 // Tmsg PDU
  rc:       P9Fcall,                 // Rmsg PDU
  wq:       WaitQueue,
  status:   AtomicI32,               // REQ_STATUS_*
  t_err:    i32,
  refcount: Refcount,                // SLAB_TYPESAFE_BY_RCU
  req_list: ListHead,
}

struct P9Fcall {
  sdata:    *mut u8,
  size:     u32,
  capacity: u32,
  offset:   u32,
  tag:      u16,
  id:       u8,
  zc:       bool,
  cache:    Option<&KmemCache>,      // fcall_cache if hot-path
}

struct P9Fid {
  clnt:    *mut P9Client,
  fid:     u32,            // server-side FID (< P9_NOFID)
  qid:     P9Qid,
  mode:    i32,
  iounit:  u32,
  uid:     KUid,
  count:   Refcount,
  rdir:    Option<P9DirContext>,
}

struct P9TransModule {
  name:               &'static CStr,
  maxsize:            i32,
  pooled_rbuffers:    bool,
  supports_vmalloc:   bool,
  def:                i32,
  owner:              &'static Module,
  create:             fn(&mut P9Client, &FsContext) -> Result<()>,
  close:              fn(&mut P9Client),
  request:            fn(&P9Client, &mut P9Req) -> i32,
  cancel:             fn(&P9Client, &mut P9Req) -> i32,
  cancelled:          fn(&P9Client, &mut P9Req) -> i32,
  zc_request:         fn(&P9Client, &mut P9Req, &mut IovIter, &mut IovIter, i32, i32, i32) -> i32,
  show_options:       fn(&SeqFile, &P9Client) -> i32,
}
```

`P9Client::rpc(c, type, fmt, args) -> Result<P9Req>`:
1. `tsize = 0; rsize = if c.trans_mod.pooled_rbuffers { c.msize } else { 0 }`.
2. `req = P9Client::prepare_req(c, type, tsize, rsize, fmt, args)?`.
3. `req.tc.zc = false; req.rc.zc = false`.
4. `sigpending = signal_pending(current); if sigpending { clear_thread_flag(TIF_SIGPENDING) }`.
5. `err = c.trans_mod.request(c, &req)`.
6. if `err < 0`:
   - `P9Client::req_put(c, &req)`.
   - if `err != -ERESTARTSYS && err != -EFAULT`: `c.status = Disconnected`.
   - goto recalc.
7. loop `again`: `err = io_wait_event_killable(req.wq, READ_ONCE(req.status) >= REQ_STATUS_RCVD)`.
8. `smp_rmb()`.
9. if `err == -ERESTARTSYS && c.status == Connected && type == P9_TFLUSH`: re-mask sigpending; `goto again`.
10. if `READ_ONCE(req.status) == REQ_STATUS_ERROR`: `err = req.t_err`.
11. if `err == -ERESTARTSYS && c.status == Connected`:
    - re-mask sigpending.
    - if `c.trans_mod.cancel(c, &req)`: `P9Client::flush(c, &req)`.
    - if `READ_ONCE(req.status) == REQ_STATUS_RCVD`: `err = 0`.
12. recalc: if `sigpending`: under `siglock`, `recalc_sigpending()`.
13. if `err < 0`: `P9Client::req_put(c, &req); return Err(safe_errno(err))`.
14. `P9Client::check_errors(c, &req)?`.
15. return `Ok(req)`.

`P9Client::zc_rpc(c, type, uidata, uodata, inlen, olen, in_hdrlen, fmt, args)`:
1. `req = P9Client::prepare_req(c, type, P9_ZC_HDR_SZ, P9_ZC_HDR_SZ, fmt, args)?`.
2. `req.tc.zc = true; req.rc.zc = true`.
3. Mask sigpending as in `rpc`.
4. `err = c.trans_mod.zc_request(c, &req, uidata, uodata, inlen, olen, in_hdrlen)`.
5. if `err == -EIO`: `c.status = Disconnected`.
6. Same response-status handling as `rpc` (incl. flush-on-interrupt).
7. `P9Client::check_errors(c, &req)?`.

`P9Client::version(c)`:
1. version-string ← dialect.
2. `req = P9Client::rpc(c, P9_TVERSION, "ds", c.msize, version_string)?`.
3. `p9pdu_readf(&req.rc, c.proto_version, "ds", &msize, &server_version)`.
4. dispatch on prefix; reject unknown with `-EREMOTEIO`.
5. reject `msize < 4096` with `-EREMOTEIO`.
6. `c.msize = min(c.msize, msize)`.

`P9Client::create(fc)`:
1. `clnt = alloc; init lock, fids, reqs idrs`.
2. `P9Client::apply_options(clnt, fc)`.
3. `clnt.trans_mod = clnt.trans_mod.or_else(|| P9Trans::default())`.
4. `clnt.trans_mod.ok_or(-EPROTONOSUPPORT)?`.
5. `(clnt.trans_mod.create)(clnt, fc)?` — establishes wire connection.
6. clamp `clnt.msize` to `trans_mod.maxsize`; reject `< 4096`.
7. `P9Client::version(clnt)?`.
8. Allocate `fcall_cache` slab.
9. Return clnt.

`P9Client::attach(clnt, afid, uname, n_uname, aname)`:
1. `fid = P9Client::fid_create(clnt)?`.
2. `fid.uid = n_uname`.
3. `req = P9Client::rpc(clnt, P9_TATTACH, "ddss?u", fid.fid, afid.map_or(P9_NOFID, |a| a.fid), uname, aname, n_uname)?`.
4. `p9pdu_readf(&req.rc, .., "Q", &qid)`.
5. `fid.qid = qid`.

`P9Client::walk(oldfid, nwname, wnames, clone)`:
1. `fid = if clone { P9Client::fid_create(clnt)?; fid.uid = oldfid.uid } else { oldfid }`.
2. `req = P9Client::rpc(clnt, P9_TWALK, "ddT", oldfid.fid, fid.fid, nwname, wnames)?`.
3. `p9pdu_readf(&req.rc, .., "R", &nwqids, &wqids)`.
4. if `nwqids != nwname`: clunk_fid → `-ENOENT`.
5. `fid.qid = wqids.last()` (or `oldfid.qid` when `nwname == 0`).

`P9Client::cb(c, req, status)`:
1. `smp_wmb()`.
2. `WRITE_ONCE(req.status, status)`.
3. `wake_up(&req.wq)`.
4. `P9Client::req_put(c, req)`.

### Out of Scope

- `net/9p/protocol.c` (PDU marshalling helpers `p9pdu_readf`/`p9pdu_writef`/`p9pdu_prepare`/`p9pdu_finalize`; covered briefly in `p9.md`)
- `net/9p/error.c` (`p9_errstr2errno` table; covered briefly in `p9.md`)
- `net/9p/trans_*.c` per-transport bodies (each merits its own Tier-3 if expanded)
- `fs/9p/*` VFS-layer adapter (separate Tier-3)
- 9P server side (not in kernel)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct p9_client` | per-mount client state (`trans_mod`, `msize`, `proto_version`, `fids`, `reqs`, `lock`, `fcall_cache`, `status`) | `P9Client` |
| `struct p9_req_t` | per-request (tag, `tc` Tmsg pdu, `rc` Rmsg pdu, `wq`, refcount, status) | `P9Req` |
| `struct p9_fcall` | per-pdu wire buffer (`sdata`, `size`, `capacity`, `offset`, `tag`, `id`, `zc`, `cache`) | `P9Fcall` |
| `struct p9_fid` | per-FID (4-byte server handle + uid + mode + qid) | `P9Fid` |
| `struct p9_qid` | per-server-side identity (type+version+path) | `P9Qid` |
| `struct p9_trans_module` | per-transport vtable | `P9TransModule` |
| `p9_client_create()` | per-mount: alloc + parse opts + resolve trans + `Tversion` | `P9Client::create` |
| `p9_client_destroy()` | per-unmount tear-down | `P9Client::destroy` |
| `p9_client_disconnect()` | per-status → `Disconnected` | `P9Client::disconnect` |
| `p9_client_begin_disconnect()` | per-only-`Tclunk`-allowed phase | `P9Client::begin_disconnect` |
| `p9_client_version()` | per-`Tversion`/`Rversion` handshake + msize-cap | `P9Client::version` |
| `p9_client_attach()` | per-`Tattach`: bind FID to fileserver root | `P9Client::attach` |
| `p9_client_walk()` | per-`Twalk`: traverse path; clone or reuse FID | `P9Client::walk` |
| `p9_client_open()` | per-`Topen`/`Tlopen` | `P9Client::open` |
| `p9_client_fcreate()` | per-`Tcreate` (legacy/.u) | `P9Client::fcreate` |
| `p9_client_create_dotl()` | per-`Tlcreate` (.L) | `P9Client::create_dotl` |
| `p9_client_read()` / `read_once()` | per-`Tread` iter loop | `P9Client::read` / `read_once` |
| `p9_client_write()` / `write_subreq()` | per-`Twrite` (incl. netfs subreq) | `P9Client::write` / `write_subreq` |
| `p9_client_clunk()` | per-`Tclunk` (release FID server-side) | `P9Client::clunk` |
| `p9_client_remove()` | per-`Tremove` | `P9Client::remove` |
| `p9_client_unlinkat()` | per-`Tunlinkat` (.L) | `P9Client::unlinkat` |
| `p9_client_stat()` / `getattr_dotl()` | per-`Tstat`/`Tgetattr` | `P9Client::stat` / `getattr_dotl` |
| `p9_client_wstat()` / `setattr()` | per-`Twstat`/`Tsetattr` | `P9Client::wstat` / `setattr` |
| `p9_client_statfs()` | per-`Tstatfs` (.L) | `P9Client::statfs` |
| `p9_client_rename()` / `renameat()` | per-`Trename`/`Trenameat` | `P9Client::rename` / `renameat` |
| `p9_client_readdir()` | per-`Treaddir` (.L) | `P9Client::readdir` |
| `p9_client_mknod_dotl()` / `mkdir_dotl()` / `symlink()` / `link()` / `readlink()` | per-namespace ops (.L) | `P9Client::mknod_dotl` / `mkdir_dotl` / `symlink` / `link` / `readlink` |
| `p9_client_xattrwalk()` / `xattrcreate()` | per-`Txattrwalk`/`Txattrcreate` (.L) | `P9Client::xattrwalk` / `xattrcreate` |
| `p9_client_lock_dotl()` / `getlock_dotl()` | per-byte-range lock | `P9Client::lock_dotl` / `getlock_dotl` |
| `p9_client_fsync()` | per-`Tfsync` (.L) | `P9Client::fsync` |
| `p9_client_rpc()` | per-blocking request/response | `P9Client::rpc` |
| `p9_client_zc_rpc()` | per-zero-copy request/response (read/write) | `P9Client::zc_rpc` |
| `p9_client_prepare_req()` | per-marshal Tmsg into `req->tc` | `P9Client::prepare_req` |
| `p9_client_flush()` | per-`Tflush` to cancel in-flight | `P9Client::flush` |
| `p9_client_cb()` | per-transport-rx callback (status + wakeup + put) | `P9Client::cb` |
| `p9_tag_alloc()` | per-`Treq` allocation, `idr_alloc` reqs | `P9Client::tag_alloc` |
| `p9_tag_lookup()` | per-rcu tag → req lookup (`SLAB_TYPESAFE_BY_RCU`) | `P9Client::tag_lookup` |
| `p9_tag_remove()` | per-`idr_remove(reqs, tag)` | `P9Client::tag_remove` |
| `p9_tag_cleanup()` | per-destroy: drain leftover reqs | `P9Client::tag_cleanup` |
| `p9_req_put()` | per-refcount dec + free | `P9Client::req_put` |
| `p9_fid_create()` / `_destroy()` | per-FID idr alloc/free | `P9Client::fid_create` / `fid_destroy` |
| `p9_fcall_init()` / `_fini()` | per-pdu buffer alloc (`fcall_cache`/`kvmalloc`/`kmalloc`) | `P9Fcall::init` / `fini` |
| `p9_check_errors()` | per-`Rerror`/`Rlerror` → `-errno` mapping | `P9Client::check_errors` |
| `p9_parse_header()` | per-pdu: read `size,type,tag` | `P9Pdu::parse_header` |
| `p9_show_client_options()` | per-`/proc/mounts` showing | `P9Client::show_options` |
| `apply_client_options()` | per-`fs_context` → client | `P9Client::apply_options` |
| `safe_errno()` | per-bound-check `-errno` | helper |
| `p9_is_proto_dotl()` / `_dotu()` | per-dialect predicate | `P9Client::is_dotl` / `is_dotu` |
| `v9fs_register_trans()` (mod.c) | per-trans-module registration | `P9Trans::register` |
| `v9fs_get_trans_by_name()` (mod.c) | per-name lookup | `P9Trans::by_name` |
| `v9fs_get_default_trans()` (mod.c) | per-default fallback (`virtio` then `tcp`) | `P9Trans::default` |
| `v9fs_put_trans()` (mod.c) | per-module ref-put | `P9Trans::put` |
| `p9_req_cache` (KMEM_CACHE, `SLAB_TYPESAFE_BY_RCU`) | per-req slab | `P9_REQ_CACHE` |
| `p9_client_init()` / `_exit()` | per-module init/exit | `P9Client::module_init` / `module_exit` |

### compatibility contract

REQ-1: `struct p9_client`:
- `lock: spinlock_t` — guards `fids` and `reqs` idrs.
- `msize: u32` — max message size (negotiated; lower bound 4096; capped by `trans_mod->maxsize`).
- `proto_version: enum p9_proto_versions` — `p9_proto_legacy` (`9P2000`), `p9_proto_2000u` (`9P2000.u`), `p9_proto_2000L` (`9P2000.L`).
- `trans_mod: *p9_trans_module` — bound transport (ref'd; `v9fs_put_trans` on destroy).
- `status: enum p9_trans_status` — `Connected` / `BeginDisconnect` (only `Tclunk` allowed) / `Disconnected`.
- `fids: idr` — 32-bit FIDs in `[0, P9_NOFID − 1]`.
- `reqs: idr` — 16-bit tags in `[0, P9_NOTAG)`; reserved `P9_NOTAG` (= `~0u16`) used only for `Tversion`.
- `fcall_cache: *kmem_cache` — per-client slab sized to `msize` for hot-path Tmsg/Rmsg buffers (`kmem_cache_create_usercopy`, allows usercopy `[P9_HDRSZ+4 .. msize]`).
- `trans: *void` — transport's per-client state.
- `name[]` — node-name (`utsname()->nodename`) for diagnostics.

REQ-2: `struct p9_req_t`:
- `tc: p9_fcall` — Tmsg PDU (transport sender consumes; size pre-finalized).
- `rc: p9_fcall` — Rmsg PDU (transport receiver fills).
- `wq: wait_queue_head_t` — completion wait.
- `status: int` — `REQ_STATUS_ALLOC` / `REQ_STATUS_UNSENT` / `REQ_STATUS_SENT` / `REQ_STATUS_RCVD` / `REQ_STATUS_FLSHD` / `REQ_STATUS_ERROR`.
- `t_err: int` — transport-level error (`-errno`).
- `refcount: refcount_t` — initialized to 0, set to 2 after `idr_alloc` succeeds (writer ref + tag-lookup-recv ref; `p9_client_cb` puts the recv ref).
- Slab: `p9_req_cache = KMEM_CACHE(p9_req_t, SLAB_TYPESAFE_BY_RCU)` — req objects are safe to read concurrently under RCU.

REQ-3: `p9_tag_alloc(c, type, t_size, r_size, fmt, ap)`:
- `req = kmem_cache_alloc(p9_req_cache, GFP_NOFS)`.
- `alloc_tsize = min(c.msize, t_size ?: p9_msg_buf_size(c, type, fmt, ap))`; same for `alloc_rsize` with `type+1`.
- `p9_fcall_init(c, &req.tc, alloc_tsize)` and `&req.rc, alloc_rsize`.
- `p9pdu_reset(&req.tc); p9pdu_reset(&req.rc)`.
- `req.t_err = 0; req.status = REQ_STATUS_ALLOC; refcount_set(req.refcount, 0)`.
- `init_waitqueue_head(&req.wq); INIT_LIST_HEAD(&req.req_list)`.
- `idr_preload(GFP_NOFS); spin_lock_irq(&c.lock)`.
- `if type == P9_TVERSION: tag = idr_alloc(&c.reqs, req, P9_NOTAG, P9_NOTAG+1, GFP_NOWAIT)` else `idr_alloc(&c.reqs, req, 0, P9_NOTAG, GFP_NOWAIT)`.
- `req.tc.tag = tag; spin_unlock_irq; idr_preload_end`.
- `refcount_set(req.refcount, 2)`.

REQ-4: `p9_tag_lookup(c, tag)`:
- `rcu_read_lock`.
- `again: req = idr_find(&c.reqs, tag)`.
- if req: `if !p9_req_try_get(req): goto again`. `if req.tc.tag != tag: p9_req_put(c, req); goto again`.
- `rcu_read_unlock`. Returns req with one extra ref or NULL.

REQ-5: `p9_req_put(c, r)`:
- if `refcount_dec_and_test(&r.refcount)`: `p9_tag_remove(c, r); p9_fcall_fini(&r.tc); p9_fcall_fini(&r.rc); kmem_cache_free(p9_req_cache, r); return 1`. Else return 0.

REQ-6: `p9_client_cb(c, req, status)`:
- `smp_wmb()` — make req body visible to other thread before status change.
- `WRITE_ONCE(req.status, status)`.
- `wake_up(&req.wq)`.
- `p9_req_put(c, req)` — release the recv-side ref.

REQ-7: `p9_parse_header(pdu, *size, *type, *tag, rewind)`:
- Read `s32 r_size`, `s8 r_type`, `s16 r_tag` from `pdu`.
- If `rewind`: reset `pdu.offset` to entry.
- Out params optional (NULL skips).

REQ-8: `p9_check_errors(c, req)`:
- `p9_parse_header(&req.rc, NULL, &type, NULL, 0)`.
- if `req.rc.size > req.rc.capacity && !req.rc.zc`: return `-EIO` (oversize).
- `trace_9p_protocol_dump(c, &req.rc)`.
- if `type != P9_RERROR && type != P9_RLERROR`: return 0.
- if `p9_is_proto_dotl(c)`: read `d ecode`; return `-ecode` (RLERROR).
- else: read `s ename` then optional `d ecode` (.u); if `.u && ecode < 512`: `err = -ecode` else `err = p9_errstr2errno(ename, len)`.

REQ-9: `p9_client_prepare_req(c, type, t_size, r_size, fmt, ap)`:
- if `c.status == Disconnected`: return `-EIO`.
- if `c.status == BeginDisconnect && type != P9_TCLUNK`: return `-EIO`.
- `req = p9_tag_alloc(c, type, t_size, r_size, fmt, ap_copy)`.
- `p9pdu_prepare(&req.tc, req.tc.tag, type)`.
- `p9pdu_vwritef(&req.tc, c.proto_version, fmt, ap)`.
- `p9pdu_finalize(c, &req.tc)`.

REQ-10: `p9_client_rpc(c, type, fmt, ...)`:
- `tsize = 0`; `rsize = (c.trans_mod.pooled_rbuffers ? c.msize : 0)`.
- `req = p9_client_prepare_req(c, type, tsize, rsize, fmt, ap)`.
- `req.tc.zc = false; req.rc.zc = false`.
- /* Mask `TIF_SIGPENDING` while issuing so signal does not corrupt wire-state */
- `err = c.trans_mod.request(c, req)`.
- if `err < 0`: `p9_req_put(c, req)`; if `err != -ERESTARTSYS && err != -EFAULT`: `c.status = Disconnected`; fall through to recalc.
- `again: err = io_wait_event_killable(req.wq, READ_ONCE(req.status) >= REQ_STATUS_RCVD)`.
- `smp_rmb()` (paired with `p9_client_cb`'s `smp_wmb`).
- if `err == -ERESTARTSYS && c.status == Connected && type == P9_TFLUSH`: re-mask + `goto again` (cannot interrupt a Tflush).
- if `READ_ONCE(req.status) == REQ_STATUS_ERROR`: `err = req.t_err`.
- if `err == -ERESTARTSYS && c.status == Connected`: if `c.trans_mod.cancel(c, req)`: `p9_client_flush(c, req)`; if response arrived anyway: clear err.
- recalc TIF_SIGPENDING under siglock if we masked.
- `p9_check_errors(c, req)`.
- on success: return req (caller does `p9_req_put` after consuming `req.rc`).

REQ-11: `p9_client_zc_rpc(c, type, uidata, uodata, inlen, olen, in_hdrlen, fmt, ...)`:
- `req = p9_client_prepare_req(c, type, P9_ZC_HDR_SZ, P9_ZC_HDR_SZ, fmt, ap)` — only header buffers; payload goes zero-copy.
- `req.tc.zc = true; req.rc.zc = true`.
- `c.trans_mod.zc_request(c, req, uidata, uodata, inlen, olen, in_hdrlen)` — transport pins user pages / passes `iov_iter` and may DMA in/out directly.
- Same wait-and-cancel logic as `p9_client_rpc`.

REQ-12: `p9_client_flush(c, oldreq)`:
- `p9_parse_header(&oldreq.tc, NULL, NULL, &oldtag, 1)`.
- `req = p9_client_rpc(c, P9_TFLUSH, "w", oldtag)`.
- if `READ_ONCE(oldreq.status) == REQ_STATUS_SENT` and `c.trans_mod.cancelled`: `c.trans_mod.cancelled(c, oldreq)`.
- `p9_req_put(c, req)`.

REQ-13: `p9_client_version(c)`:
- per `c.proto_version` send `Tversion`:
  - `p9_proto_2000L`: `"9P2000.L"`.
  - `p9_proto_2000u`: `"9P2000.u"`.
  - `p9_proto_legacy`: `"9P2000"`.
- `req = p9_client_rpc(c, P9_TVERSION, "ds", c.msize, version_str)`.
- `p9pdu_readf(&req.rc, c.proto_version, "ds", &msize, &version)`.
- Parse server `version`:
  - prefix `"9P2000.L"` → `c.proto_version = p9_proto_2000L`.
  - prefix `"9P2000.u"` → `c.proto_version = p9_proto_2000u`.
  - prefix `"9P2000"` → `c.proto_version = p9_proto_legacy`.
  - else `-EREMOTEIO`.
- if `msize < 4096`: `-EREMOTEIO`.
- if `msize < c.msize`: `c.msize = msize` (server-imposed cap).

REQ-14: `p9_client_create(fc)`:
1. `clnt = kmalloc_obj(p9_client)`; init `lock`, `idr_init(&clnt.fids)`, `idr_init(&clnt.reqs)`.
2. Copy `nodename` into `clnt.name`.
3. `apply_client_options(clnt, fc)` — pull `msize`, `trans_mod`, `proto_version` from `v9fs_context` (set by mount-option parser).
4. if `!clnt.trans_mod`: `clnt.trans_mod = v9fs_get_default_trans()` (first registered default — typically `virtio` then `tcp`).
5. if `!clnt.trans_mod`: `-EPROTONOSUPPORT`.
6. `clnt.trans_mod.create(clnt, fc)` — per-transport connect.
7. if `clnt.msize > clnt.trans_mod.maxsize`: `clnt.msize = clnt.trans_mod.maxsize` (with log).
8. if `clnt.msize < 4096`: `-EINVAL`.
9. `p9_client_version(clnt)` — negotiate with server.
10. Allocate `fcall_cache` = `kmem_cache_create_usercopy("9p-fcall-cache-%u", msize, 0, 0, P9_HDRSZ+4, msize-(P9_HDRSZ+4), NULL)` — bounds usercopy slab regions.
11. Failure cleans up in order: `close_trans` → `put_trans` → `free_client`.

REQ-15: `p9_client_destroy(clnt)`:
- if `trans_mod`: `trans_mod.close(clnt)`.
- `v9fs_put_trans(trans_mod)`.
- `idr_for_each_entry(&clnt.fids, fid, id)`: log "Found fid not clunked"; `p9_fid_destroy(fid)`.
- `p9_tag_cleanup(clnt)` — RCU-walk leftover reqs, `p9_req_put` each.
- `kmem_cache_destroy(fcall_cache); kfree(clnt)`.

REQ-16: `p9_client_attach(clnt, afid, uname, n_uname, aname)`:
- `fid = p9_fid_create(clnt)` (allocates FID via `idr_alloc_u32(&clnt.fids, ..., P9_NOFID-1)`).
- `fid.uid = n_uname`.
- `req = p9_client_rpc(clnt, P9_TATTACH, "ddss?u", fid.fid, afid ? afid.fid : P9_NOFID, uname, aname, n_uname)` (the `?u` carries `n_uname` only in `.u`/.L dialects).
- `p9pdu_readf(&req.rc, "Q", &qid)`; copy qid into `fid.qid`.

REQ-17: `p9_client_walk(oldfid, nwname, wnames, clone)`:
- if `clone`: `fid = p9_fid_create(clnt); fid.uid = oldfid.uid` (new server-side FID for walk result); else reuse `oldfid`.
- `req = p9_client_rpc(clnt, P9_TWALK, "ddT", oldfid.fid, fid.fid, nwname, wnames)`.
- `p9pdu_readf(&req.rc, "R", &nwqids, &wqids)`.
- if `nwqids != nwname`: partial walk → `-ENOENT` (clunk_fid path).
- final qid copied to `fid.qid`.
- Error path: clunk_fid → `p9_fid_put(fid); if fid != oldfid: p9_fid_destroy(fid)`.

REQ-18: `p9_fid_create(clnt)`:
- `fid = kzalloc_obj(p9_fid); fid.mode = -1; fid.uid = current_fsuid(); fid.clnt = clnt; refcount_set(&fid.count, 1)`.
- `idr_preload(GFP_KERNEL); spin_lock_irq(&clnt.lock); idr_alloc_u32(&clnt.fids, fid, &fid.fid, P9_NOFID-1, GFP_NOWAIT)`.
- On success: `trace_9p_fid_ref(fid, P9_FID_REF_CREATE)`. On fail: `kfree(fid)`.

REQ-19: `struct p9_trans_module` (from `include/net/9p/transport.h`):
- `name: *const char` — matched at mount (`trans=` option).
- `maxsize: int` — transport-imposed msize cap.
- `pooled_rbuffers: bool` — set by RDMA; forces `rsize = c.msize` in `p9_client_rpc` (cannot trim).
- `supports_vmalloc: bool` — `p9_fcall_init` may fall back to `kvmalloc`.
- `def: int` — default-rank.
- `owner: *module`.
- Vtable: `create(client, fs_context) → int`; `close(client) → void`; `request(client, req) → int`; `cancel(client, req) → int`; `cancelled(client, req) → int`; `zc_request(client, req, uidata, uodata, inlen, olen, in_hdrlen) → int`; `show_options(seq, client) → int`.
- Registry list: linked into `v9fs_trans_list` by `v9fs_register_trans`; protected by `v9fs_trans_lock` (in `mod.c`). Refcount via `try_module_get(m->owner)` in `v9fs_get_trans_by_name`; released by `v9fs_put_trans(m) → module_put(m->owner)`.

REQ-20: Registered transports at module init:
- `trans_fd.c`: registers `p9_tcp_trans` (`name="tcp"`), `p9_unix_trans` (`name="unix"`, AF_UNIX stream), `p9_fd_trans` (`name="fd"`, pre-opened rfd/wfd pair).
- `trans_virtio.c`: registers `p9_virtio_trans` (`name="virtio"`, virtio-9p / 9P/RPC over virtqueues).
- `trans_rdma.c`: registers `p9_rdma_trans` (`name="rdma"`, ib_verbs SEND/RECV; `pooled_rbuffers=true`).
- `trans_xen.c`: registers `p9_xen_trans` (`name="xen"`, xen-9pfs over xenbus rings).
- `trans_usbg.c`: registers `p9_usbg_trans` (USB gadget) when configured.

REQ-21: Error mapping:
- `safe_errno(err)` — clamps result to `[-MAX_ERRNO, 0]`; else `-EPROTO` and log "Invalid error code".
- `p9_errstr2errno(name, len)` (in `error.c`) — translates Plan 9 string error to host errno.

REQ-22: Disconnect / flush invariants:
- `c.status` transitions:
  - `Connected` → `BeginDisconnect` (via `p9_client_begin_disconnect`): only `Tclunk` allowed.
  - `Connected` → `Disconnected` (via `p9_client_disconnect` or `trans_mod.request` fatal err): no requests allowed (`-EIO`).
- A `Tflush` itself is never cancelled by a signal — `p9_client_rpc` loops over `io_wait_event_killable` masking sigpending while the flush is in flight.

REQ-23: FID lifetimes:
- Server-side: server tracks FID until `Tclunk` (or `Tremove`).
- Client-side: `p9_fid` is refcounted (`refcount_t count`); `p9_fid_get`/`_put` traced via tracepoint `9p_fid_ref`.
- Final `p9_fid_put` triggers `p9_client_clunk` then `p9_fid_destroy` (removes from `clnt.fids` idr and frees).

REQ-24: `Twrite` paths:
- `p9_client_write(fid, offset, from_iter, *err)` — splits writes by `rsize = min(iov_iter_count, fid.iounit, c.msize - P9_IOHDRSZ)`, uses `p9_client_zc_rpc(P9_TWRITE, NULL, from_iter, 0, rsize, P9_ZC_HDR_SZ, "dqd", ...)` for zero-copy.
- `p9_client_write_subreq(netfs_subreq)` — netfs-glue path: per-subrequest sliced writes, completes subreq with bytes-written.

REQ-25: `Tread` paths:
- `p9_client_read(fid, offset, to_iter, *err)` — loops calling `p9_client_read_once` until `to_iter` drained or short-read.
- `p9_client_read_once` issues `P9_TREAD` via `p9_client_zc_rpc(.., uidata=to_iter, uodata=NULL, inlen=rsize, in_hdrlen=11)`.

REQ-26: Module init: `p9_client_init() → p9_req_cache = KMEM_CACHE(p9_req_t, SLAB_TYPESAFE_BY_RCU); return p9_req_cache ? 0 : -ENOMEM`. Exit: `kmem_cache_destroy(p9_req_cache)`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rpc_tag_idr_bounded` | INVARIANT | per-`tag_alloc`: returned tag ∈ `[0, P9_NOTAG)` ∨ `tag == P9_NOTAG` iff `type == P9_TVERSION`. |
| `req_refcount_balanced` | INVARIANT | per-req lifecycle: writer-put + cb-put + main-put == 0 references; no UAF. |
| `slab_typesafe_by_rcu` | INVARIANT | per-`p9_tag_lookup`: even if `idr_find` returns a re-allocated req, mismatched-tag retry guarantees correctness. |
| `disconnected_rejects_request` | INVARIANT | per-`prepare_req`: `status == Disconnected` ⟹ `-EIO`. |
| `begin_disconnect_only_clunk` | INVARIANT | per-`prepare_req`: `status == BeginDisconnect && type != P9_TCLUNK` ⟹ `-EIO`. |
| `msize_lower_bound` | INVARIANT | per-`create` and per-`version`: `c.msize >= 4096`. |
| `tversion_uses_notag` | INVARIANT | per-`tag_alloc`: `type == P9_TVERSION` ⟹ tag == `P9_NOTAG`. |
| `safe_errno_in_range` | INVARIANT | per-`safe_errno`: returned err ∈ `[-MAX_ERRNO, 0]` ∨ `-EPROTO`. |
| `fcall_size_le_capacity` | INVARIANT | per-`check_errors`: `rc.size > rc.capacity && !zc` ⟹ `-EIO`. |
| `cb_wmb_before_status` | INVARIANT | per-`client_cb`: `smp_wmb` precedes `WRITE_ONCE(status)` — pairs with `rpc`'s `smp_rmb`. |

### Layer 2: TLA+

`net/9p-rpc.tla`:
- Per-Tversion handshake + per-tagged-request + per-flush + per-disconnect.
- Properties:
  - `safety_unique_tag` — per-tag-alloc: no two concurrent reqs share a non-`P9_NOTAG` tag.
  - `safety_response_matches_tag` — per-cb: `req.tc.tag == rc_tag`.
  - `safety_flush_cancels_or_completes` — per-`Tflush`: either `cancelled` callback fires or `Rmsg` arrives.
  - `safety_disconnected_no_new` — per-`Disconnected`: no new req is accepted.
  - `liveness_rpc_terminates` — per-`io_wait_event_killable`: either response received, transport cancel fires, or signal aborts.
  - `liveness_destroy_drains` — per-`p9_client_destroy`: every leftover req in `c.reqs` is freed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `P9Client::rpc` post: returns either `Ok(req)` with `rc.id` set, or `Err(-errno)` with no leaked req | `P9Client::rpc` |
| `P9Client::zc_rpc` post: payload not copied through `req.rc.sdata`; `req.rc.zc == true` | `P9Client::zc_rpc` |
| `P9Client::version` post: `proto_version ∈ {legacy, 2000u, 2000L}` ∧ `msize ≥ 4096` | `P9Client::version` |
| `P9Client::create` post: `trans_mod != NULL` ∧ `fcall_cache != NULL` ∧ version-negotiated | `P9Client::create` |
| `P9Client::walk` post: clone-FID either bound to new server-FID with qid set, or destroyed | `P9Client::walk` |
| `P9Client::attach` post: returned FID's `qid` matches `Rattach` payload | `P9Client::attach` |
| `P9Client::tag_alloc` post: req refcount == 2 ∧ inserted in `c.reqs` ∧ tag in valid range | `P9Client::tag_alloc` |
| `P9Client::cb` post: `status` written; `wake_up` issued; one ref released | `P9Client::cb` |
| `P9Client::check_errors` post: returns 0 only when `type ∉ {RERROR, RLERROR}` | `P9Client::check_errors` |
| `P9Client::flush` post: `Tflush` issued and consumed; `oldreq` cancelled-or-completed | `P9Client::flush` |

### Layer 4: Verus/Creusot functional

`Per-mount: create → trans_mod.create → version → attach root → walk … → open → (read|write|...) → clunk → unmount → destroy` semantic equivalence with upstream `9pfs` against `Documentation/filesystems/9p.rst` and the Plan 9 protocol manual. Per-tag namespace, per-dialect parsing rules (legacy / `.u` / `.L`), per-`Rerror`/`Rlerror` error mapping, and per-`Tflush` cancellation semantics must match upstream byte-for-byte on the wire.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

9P client reinforcement:

- **Per-tag space exhausted ⇒ `-ENOMEM`** — defense against per-pending-request flood.
- **Per-`Tversion` reserved tag (`P9_NOTAG`)** — defense against per-version-collision with normal RPCs.
- **Per-`msize ≥ 4096` lower-bound** — defense against per-server-degenerate negotiation.
- **Per-`trans_mod.maxsize` cap** — defense against per-trans-buffer overflow.
- **Per-`fcall_cache` usercopy region `[P9_HDRSZ+4 .. msize]`** — defense against per-out-of-region usercopy.
- **Per-`SLAB_TYPESAFE_BY_RCU` + tag re-check** — defense against per-tag-recycled UAF in `p9_tag_lookup`.
- **Per-`safe_errno` clamp** — defense against per-server-malicious errno (`< -MAX_ERRNO` or `> 0`).
- **Per-`Rerror`/`Rlerror` dialect-strict parsing** — defense against per-payload-confusion across `.L` vs `.u` vs legacy.
- **Per-`rc.size > rc.capacity` ⇒ `-EIO`** — defense against per-oversized-reply buffer overrun.
- **Per-`status = Disconnected` short-circuit** — defense against per-zombie-mount continuing RPCs after fatal transport error.
- **Per-`BeginDisconnect` only `Tclunk`** — defense against per-shutdown FID leak.
- **Per-`Tflush` not interruptible** — defense against per-Tflush-storm from signal racing.
- **Per-FID `count` refcount + tracepoints** — defense against per-FID-leak (visible via `9p_fid_ref` ftrace).
- **Per-`module_get`/`module_put` on `trans_mod`** — defense against per-module unload while mount active.

