---
title: "Tier-3: fs/ceph/super.c — Ceph fs client superblock + mount"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`fs/ceph/super.c` is the **VFS-facing entry point** for the Ceph distributed filesystem client. It owns: (a) the `ceph_fs_client` per-superblock state (ceph monc/osdc handles, mdsc, mount-options, fscache hooks, write-congestion gauges, async-unlink hash), (b) the `ceph_mount_options` parser (fs_parameter_spec table, old `mon_ips:/path` vs. new `name@fsid.fsname=/path` device syntax), (c) the `fill_super` / `ceph_get_tree` flow that brings up libceph mon/osd, requests `MDSMAP` or `FSMAP`, opens a session, walks to the export root and pins a root dentry, (d) the umount/kill_sb teardown that flushes dirty caps, waits on stopping-blockers, drains workqueues, and unwinds the libceph client, plus (e) cross-mount sharing via `compare_super`. Per-libceph: this file is the GLUE between Linux VFS and the libceph protocol stack (mon_client / osd_client / auth / messenger).

This Tier-3 covers `fs/ceph/super.c` (~1716 lines).

### Acceptance Criteria

- [ ] AC-1: `mount -t ceph mon1,mon2:/some/path …` parses via old syntax; fsopt->new_dev_syntax == false.
- [ ] AC-2: `mount -t ceph user@fsid.fsname=/path -o mon_addr=...` parses via new syntax; mds_namespace set.
- [ ] AC-3: `mount -t ceph user@fsid.fsname=/path` (no mon_addr) ⟹ -EINVAL "No monitor address".
- [ ] AC-4: `-o wsize=N` with N < PAGE_SIZE or N > CEPH_MAX_WRITE_SIZE ⟹ -EINVAL.
- [ ] AC-5: `-o recover_session=clean` sets CEPH_MOUNT_OPT_CLEANRECOVER; show_options prints "recover_session=clean".
- [ ] AC-6: `-o mds_namespace=foo,mds_namespace=bar` (mismatching) ⟹ "Mismatching mds_namespace".
- [ ] AC-7: ceph_get_tree successful: fc->root pinned, fsc->mount_state == MOUNTED, root inode has ino == CEPH_INO_ROOT.
- [ ] AC-8: Two mounts of the same fs with identical options share sb (sget_fc returns existing).
- [ ] AC-9: ceph_compare_super rejects sharing when monc/options differ, fsid differs, sb_flags differ, blocklisted-without-CLEANRECOVER, or mount_state == SHUTDOWN.
- [ ] AC-10: ceph_kill_sb waits for dirty_folios and stopping_blockers up to mount_timeout; kill_anon_super finalises.
- [ ] AC-11: ceph_force_reconnect resets blocklisted=false; refreshes root inode getattr.
- [ ] AC-12: ceph_statfs returns cluster fsid XOR-folded into f_fsid; f_type == CEPH_SUPER_MAGIC.
- [ ] AC-13: extra_mon_dispatch routes CEPH_MSG_MDS_MAP / CEPH_MSG_FS_MAP_USER to mdsc; other types ⟹ -1.
- [ ] AC-14: Without CONFIG_CEPH_FSCACHE, `-o fsc` ⟹ invalfc "fscache support is disabled".
- [ ] AC-15: `-o test_dummy_encryption` without CONFIG_FS_ENCRYPTION ⟹ warnfc only (ignored, not fatal).

### Architecture

```
struct CephFsClient {
  sb: *SuperBlock,
  client: *CephClient,                       // libceph
  mdsc: *CephMdsClient,
  mount_options: Box<CephMountOptions>,
  mount_state: CephMountState,               // MOUNTING / MOUNTED / SHUTDOWN / RECOVER
  filp_gen: AtomicU64,
  have_copy_from2: bool,
  inode_wq: *Workqueue,                      // "ceph-inode", WQ_UNBOUND
  cap_wq: *Workqueue,                        // "ceph-cap", WQ_PERCPU, max_active=1
  async_unlink_conflict: HashTable,
  async_unlink_conflict_lock: SpinLock,
  writeback_count: AtomicI64,
  write_congested: bool,
  blocklisted: bool,
  metric_wakeup: ListHead,                   // on ceph_fsc_list
  fsc_dummy_enc_policy: FscryptDummyPolicy,
}

struct CephMountOptions {
  flags: u32,                                // CEPH_MOUNT_OPT_*
  wsize: u32,                                // ALIGNed page-multiple
  rsize: u32,
  rasize: u32,
  congestion_kb: u32,
  caps_max: i32,
  caps_wanted_delay_min: u32,
  caps_wanted_delay_max: u32,
  max_readdir: u32,
  max_readdir_bytes: u32,
  snapdir_name: CString,                     // default ".snap"
  mds_namespace: Option<CString>,
  mon_addr: Option<CString>,
  server_path: Option<CString>,              // canonicalised
  fscache_uniq: Option<CString>,
  new_dev_syntax: bool,
  dummy_enc_policy: FscryptDummyPolicy,
}

enum CephMountState {
  Mounting,
  Mounted,
  Shutdown,
  Recover,
}
```

`CephSuper::init_fs_context(fc) -> Result<()>`:
1. pctx = kzalloc<CephParseOptsCtx>.
2. pctx.copts = ceph_alloc_options() (libceph defaults).
3. pctx.opts = kzalloc<CephMountOptions>.
4. fsopt.flags = CEPH_MOUNT_OPT_DEFAULT.
5. fsopt.wsize = CEPH_MAX_WRITE_SIZE; rsize = CEPH_MAX_READ_SIZE; rasize = CEPH_RASIZE_DEFAULT.
6. fsopt.snapdir_name = kstrdup(CEPH_SNAPDIRNAME_DEFAULT).
7. fsopt.caps_wanted_delay_min/max = CEPH_CAPS_WANTED_DELAY_*_DEFAULT.
8. fsopt.max_readdir = CEPH_MAX_READDIR_DEFAULT; max_readdir_bytes = CEPH_MAX_READDIR_BYTES_DEFAULT.
9. fsopt.congestion_kb = default_congestion_kb().
10. if CONFIG_CEPH_FS_POSIX_ACL: fc.sb_flags |= SB_POSIXACL.
11. fc.fs_private = pctx; fc.ops = &CEPH_CONTEXT_OPS.

`CephSuper::parse_mount_param(fc, param) -> Result<()>`:
1. /* Try libceph first */
2. ret = ceph_parse_param(param, copts, fc.log).
3. if ret != -ENOPARAM: return ret.
4. /* Fall back to ceph-fs */
5. token = fs_parse(fc, CEPH_MOUNT_PARAMS, param, &result).
6. match token: { Opt_wsize | Opt_rsize: range_check + ALIGN, Opt_recover_session: enum→flag, Opt_source: parse_source, Opt_mon_addr: parse_mon_addr, Opt_fscache: feature-gated, ... }.
7. on out-of-range: invalfc("%s out of range").

`CephSuper::parse_source(param, fc) -> Result<()>`:
1. dev_name = param.string.
2. if empty: invalfc("Empty source").
3. dev_name_end = strchr(dev_name, '/').
4. if dev_name_end: fsopt.server_path = kstrdup(dev_name_end); canonicalize_path(server_path).
5. else: dev_name_end = dev_name + strlen.
6. dev_name_end -= 1 /* back up to separator (':' or '=') */.
7. if dev_name_end < dev_name: invalfc("Path missing in source").
8. ret = parse_new_source(dev_name, dev_name_end, fc).
9. if ret == -EINVAL: ret = parse_old_source(dev_name, dev_name_end, fc).
10. fc.source = param.string; param.string = NULL.

`CephFsClient::create(fsopt, opt) -> Result<*CephFsClient>`:
1. fsc = kzalloc<CephFsClient>.
2. fsc.client = ceph_create_client(opt, fsc).
3. fsc.client.extra_mon_dispatch = extra_mon_dispatch.
4. ceph_set_opt(client, ABORT_ON_FULL).
5. if mds_namespace.is_some(): ceph_monc_want_map(monc, CEPH_SUB_FSMAP, 0, false).
6. else: ceph_monc_want_map(monc, CEPH_SUB_MDSMAP, 0, true).
7. fsc.mount_state = Mounting; filp_gen = 1; have_copy_from2 = true.
8. fsc.inode_wq = alloc_workqueue("ceph-inode", WQ_UNBOUND, 0).
9. fsc.cap_wq = alloc_workqueue("ceph-cap", WQ_PERCPU, 1).
10. hash_init(async_unlink_conflict).
11. list_add_tail(&fsc.metric_wakeup, &CEPH_FSC_LIST) under CEPH_FSC_LOCK.
12. return Ok(fsc).
13. /* Unwind on partial-failure: destroy_workqueue, ceph_destroy_client, kfree, destroy_mount_options. */

`CephSuper::real_mount(fsc, fc) -> Result<*Dentry>`:
1. started = jiffies.
2. mutex_lock(&fsc.client.mount_mutex).
3. if fsc.sb.s_root.is_none():
   - path = mount_options.server_path.map(|p| &p[1..]).unwrap_or("").
   - __ceph_open_session(fsc.client) — opens mon-session, fetches monmap+osdmap+mdsmap/fsmap, auth.
   - if FSCACHE: ceph_fscache_register_fs(fsc, fc).
   - apply_test_dummy_encryption(sb, fc, mount_options).
   - ceph_fs_debugfs_init(fsc).
   - root = open_root_dentry(fsc, path, started).
   - fsc.sb.s_root = dget(root).
4. else: root = dget(fsc.sb.s_root).
5. fsc.mount_state = Mounted.
6. mutex_unlock.

`CephSuper::open_root_dentry(fsc, path, started) -> Result<*Dentry>`:
1. req = ceph_mdsc_create_request(mdsc, CEPH_MDS_OP_GETATTR, USE_ANY_MDS).
2. req.r_path1 = kstrdup(path).
3. req.r_ino1 = (CEPH_INO_ROOT, CEPH_NOSNAP).
4. req.r_started = started; req.r_timeout = mount_timeout.
5. req.r_args.getattr.mask = CEPH_STAT_CAP_INODE.
6. req.r_num_caps = 2.
7. err = ceph_mdsc_do_request(mdsc, NULL, req).
8. if err == 0: root = d_make_root(req.r_target_inode).
9. ceph_mdsc_put_request(req); return root.

`CephSuper::get_tree(fc) -> Result<()>`:
1. if !fc.source: invalfc("No source").
2. if fsopt.new_dev_syntax ∧ fsopt.mon_addr.is_none(): invalfc("No monitor address").
3. fsc = CephFsClient::create(opts, copts) /* consumes both */.
4. ceph_mdsc_init(fsc).
5. compare_super = if ceph_test_opt(client, NOSHARE) { None } else { Some(ceph_compare_super) }.
6. fc.s_fs_info = fsc; sb = sget_fc(fc, compare_super, ceph_set_super).
7. if ceph_sb_to_fs_client(sb) != fsc: destroy_fs_client(fsc); fsc = existing.
8. else: ceph_setup_bdi(sb, fsc).
9. res = real_mount(fsc, fc).
10. on failure: if !mdsmap_is_cluster_available: err = -EHOSTUNREACH.
11. ceph_mdsc_close_sessions(fsc.mdsc); deactivate_locked_super(sb).

`CephSuper::kill_sb(s)`:
1. ceph_mdsc_pre_umount(fsc.mdsc) — cancel outstanding requests, send CLIENT_SESSION CLOSE.
2. flush_fs_workqueues(fsc) — flush inode_wq + cap_wq.
3. sync_filesystem(s) — must precede stage-bump to drop MDS-message-driven inode pins.
4. if atomic64_read(&mdsc.dirty_folios) > 0: wait_event_killable_timeout(&mdsc.flush_end_wq, dirty_folios <= 0, mount_timeout).
5. spin_lock(&mdsc.stopping_lock):
   - mdsc.stopping = CEPH_MDSC_STOPPING_FLUSHING.
   - wait = atomic_read(&mdsc.stopping_blockers) > 0.
6. if wait: wait_for_completion_killable_timeout(&mdsc.stopping_waiter, mount_timeout).
7. mdsc.stopping = CEPH_MDSC_STOPPING_FLUSHED.
8. kill_anon_super(s) — VFS finalises.
9. fsc.client.extra_mon_dispatch = NULL.
10. ceph_fs_debugfs_cleanup(fsc).
11. ceph_fscache_unregister_fs(fsc).
12. destroy_fs_client(fsc).

`CephSuper::force_reconnect(sb) -> Result<()>`:
1. fsc.mount_state = Recover.
2. __ceph_umount_begin(fsc) — ceph_osdc_abort_requests(-EIO); ceph_mdsc_force_umount; filp_gen++.
3. flush_workqueue(fsc.inode_wq).
4. ceph_reset_client_addr(fsc.client) — reseed local addr for mon/osd reconnect.
5. ceph_osdc_clear_abort_err(&fsc.client.osdc).
6. fsc.blocklisted = false; mount_state = Mounted.
7. if sb.s_root: __ceph_do_getattr(d_inode(sb.s_root), NULL, CEPH_STAT_CAP_INODE, force=true).

### Out of Scope

- fs/ceph/mds_client.c MDS-session FSM (covered in `mds-client.md` Tier-3 if expanded)
- fs/ceph/caps.c cap-state machine (covered in `caps.md` Tier-3 if expanded)
- fs/ceph/snap.c snapshot realms (covered in `snap.md` Tier-3 if expanded)
- net/ceph/mon_client.c, osd_client.c, messenger.c libceph protocol (covered in `net/ceph/*.md` Tier-3 if expanded)
- fs/ceph/crypto.c per-file encryption (covered in `crypto.md` Tier-3 if expanded)
- fs/ceph/cache.c fscache integration (covered in `cache.md` Tier-3 if expanded)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ceph_fs_client` | per-mount client state (sb, client, mdsc, mount_options, fscache, workqueues) | `CephFsClient` |
| `struct ceph_mount_options` | per-mount parsed args (wsize, rsize, mds_namespace, server_path, flags, mon_addr, fscache_uniq, snapdir_name, dummy_enc_policy) | `CephMountOptions` |
| `struct ceph_parse_opts_ctx` | per-parse fs_context private (copts + opts) | `CephParseOptsCtx` |
| `ceph_mount_parameters[]` | per-fs_parameter_spec table | `CEPH_MOUNT_PARAMS` |
| `ceph_init_fs_context()` | per-fc init: alloc pctx + opts, set defaults | `CephSuper::init_fs_context` |
| `ceph_parse_mount_param()` | per-token dispatch (libceph then ceph-fs) | `CephSuper::parse_mount_param` |
| `ceph_parse_source()` | per-device-string parser (old vs new syntax) | `CephSuper::parse_source` |
| `ceph_parse_new_source()` | per-`name@fsid.fsname=/path` parser | `CephSuper::parse_new_source` |
| `ceph_parse_old_source()` | per-`mon_ips:/path` parser | `CephSuper::parse_old_source` |
| `ceph_parse_mon_addr()` | per-`mon_addr=` parser | `CephSuper::parse_mon_addr` |
| `canonicalize_path()` | per-server_path slash-collapse | `CephSuper::canonicalize_path` |
| `create_fs_client()` | per-fsc alloc + `ceph_create_client` + monc subscribe | `CephFsClient::create` |
| `destroy_fs_client()` | per-fsc teardown | `CephFsClient::destroy` |
| `extra_mon_dispatch()` | per-mon-msg sink: CEPH_MSG_MDS_MAP / CEPH_MSG_FS_MAP_USER | `CephFsClient::extra_mon_dispatch` |
| `ceph_real_mount()` | per-mount: open_session + fscache + open_root_dentry | `CephSuper::real_mount` |
| `open_root_dentry()` | per-MDS-getattr root request | `CephSuper::open_root_dentry` |
| `ceph_set_super()` | per-sb init (s_op, s_xattr, magic, BDI, max_file_size) | `CephSuper::set_super` |
| `ceph_compare_super()` | per-sb-share predicate | `CephSuper::compare_super` |
| `compare_mount_options()` | per-option memcmp + strcmp_null + ceph_compare_options | `CephSuper::compare_mount_options` |
| `ceph_get_tree()` | per-fc top-level mount driver | `CephSuper::get_tree` |
| `ceph_reconfigure_fc()` | per-remount handler | `CephSuper::reconfigure_fc` |
| `ceph_free_fc()` | per-fc dispose | `CephSuper::free_fc` |
| `ceph_kill_sb()` | per-sb teardown: pre_umount, drain folios, stopping-blocker wait | `CephSuper::kill_sb` |
| `ceph_umount_begin()` | per-forced-umount entry | `CephSuper::umount_begin` |
| `__ceph_umount_begin()` | per-osd-abort + force_umount mdsc | `CephSuper::umount_begin_inner` |
| `ceph_force_reconnect()` | per-blocklist-recovery reconnect | `CephSuper::force_reconnect` |
| `ceph_show_options()` | per-/proc/mounts output | `CephSuper::show_options` |
| `ceph_statfs()` | per-`statfs()`: monc statfs + quota | `CephSuper::statfs` |
| `ceph_sync_fs()` | per-`sync_fs`: caps flush + osdc/mdsc sync | `CephSuper::sync_fs` |
| `ceph_put_super()` | per-`put_super`: close mds sessions | `CephSuper::put_super` |
| `ceph_setup_bdi()` | per-mount BDI w/ ra_pages, io_pages | `CephSuper::setup_bdi` |
| `ceph_apply_test_dummy_encryption()` | per-fscrypt test-mode bind | `CephSuper::apply_test_dummy_encryption` |
| `__inc_stopping_blocker / __dec_stopping_blocker` | per-mdsc-shutdown rendezvous | `CephFsClient::inc_blocker / dec_blocker` |
| `ceph_inc_mds_stopping_blocker / ceph_dec_mds_stopping_blocker` | per-MDS-IO blocker | `CephFsClient::inc_mds_blocker / dec_mds_blocker` |
| `ceph_inc_osd_stopping_blocker / ceph_dec_osd_stopping_blocker` | per-OSD-IO blocker | `CephFsClient::inc_osd_blocker / dec_osd_blocker` |
| `init_caches()` / `destroy_caches()` | per-slab-cache lifecycle | `CephSuper::init_caches / destroy_caches` |
| `ceph_fs_type` | per-file_system_type registration ("ceph") | `CEPH_FS_TYPE` |
| `ceph_super_ops` | per-super_operations vtable | `CEPH_SUPER_OPS` |
| `ceph_context_ops` | per-fs_context_operations vtable | `CEPH_CONTEXT_OPS` |

### compatibility contract

REQ-1: `struct ceph_fs_client` (per-superblock, owns libceph client):
- sb: back-pointer to `super_block`.
- client: `struct ceph_client *` (libceph mon/osd/messenger).
- mdsc: `struct ceph_mds_client *` (MDS-session manager).
- mount_options: `struct ceph_mount_options *`.
- mount_state: CEPH_MOUNT_MOUNTING / MOUNTED / SHUTDOWN / RECOVER.
- filp_gen: per-forced-umount epoch (invalidates open files).
- have_copy_from2: feature-bit cache.
- inode_wq: per-mount workqueue ("ceph-inode", WQ_UNBOUND).
- cap_wq: per-mount workqueue ("ceph-cap", WQ_PERCPU, max_active=1).
- async_unlink_conflict / _lock: per-async-unlink dedup hash + spinlock.
- writeback_count: atomic_long, BDI congestion gauge.
- write_congested: bool.
- blocklisted: bool, set when mon evicts us.
- metric_wakeup: list_head on ceph_fsc_list (per-mount tracking).
- fsc_dummy_enc_policy: fscrypt test-dummy holder.

REQ-2: `struct ceph_mount_options`:
- flags: bitmask of CEPH_MOUNT_OPT_* (DIRSTAT, RBYTES, NOASYNCREADDIR, DCACHE, INO32, FSCACHE, NOPOOLPERM, MOUNTWAIT, NOQUOTADF, NOCOPYFROM, ASYNC_DIROPS, CLEANRECOVER, NOPAGECACHE, SPARSEREAD).
- wsize / rsize / rasize: per-IO size hints (page-aligned, [PAGE_SIZE, CEPH_MAX_*]).
- congestion_kb: per-BDI congestion threshold (>= 1024).
- caps_max: caps cap (-1 = unlimited, default 0).
- caps_wanted_delay_min / _max: per-cap-renewal pacing.
- max_readdir / max_readdir_bytes: per-readdir batch.
- snapdir_name: per-snapdir name (default ".snap").
- mds_namespace: per-fs name within ceph cluster.
- mon_addr: per-monitor address list (new syntax).
- server_path: per-export root path (canonicalised).
- fscache_uniq: per-fscache unique-tag (when FSCACHE flag set).
- new_dev_syntax: bool, true for `name@fsid.fsname=/path` form.
- dummy_enc_policy: per-fscrypt test policy.

REQ-3: `ceph_mount_parameters[]` (per-fs_parameter_spec):
- u32: wsize, rsize, rasize, caps_wanted_delay_max, caps_wanted_delay_min, write_congestion_kb, readdir_max_bytes, readdir_max_entries.
- s32: caps_max.
- string: snapdirname, mds_namespace, mon_addr, source, fsc=...
- flag_no: acl, asyncreaddir, copyfrom, dcache, dirstat, fsc, ino32, poolperm, quotadf, rbytes, require_active_mds, wsync, pagecache, sparseread.
- enum: recover_session ∈ {no, clean}.
- flag/string: test_dummy_encryption.

REQ-4: `ceph_parse_source(param, fc)`:
- /* Extract optional `/<path>` suffix; canonicalize_path. */
- dev_name_end = strchr(dev_name, '/').
- if dev_name_end: server_path = kstrdup(dev_name_end); canonicalize_path.
- else: dev_name_end = dev_name + strlen.
- /* Try new syntax, fall back to old on -EINVAL */
- ret = ceph_parse_new_source(dev_name, dev_name_end, fc).
- if ret == -EINVAL: ret = ceph_parse_old_source(...).
- fc->source = param->string; param->string = NULL.

REQ-5: `ceph_parse_new_source()` (per-`name@fsid.fsname=/path`):
- separator '=' required at dev_name_end.
- fsid_start = strchr(dev_name, '@') (required).
- opts->name = kstrndup(name_start, len) — ceph entity name.
- fs_name_start = strchr(fsid_start, '.') (required).
- ceph_parse_fsid(fsid_start, &fsid).
- fsopt->mds_namespace = kstrndup(fs_name_start, fs_name_len).
- fsopt->new_dev_syntax = true.

REQ-6: `ceph_parse_old_source()` (per-`mon_ips:/path`):
- separator ':' required at dev_name_end.
- ceph_parse_mon_ips(dev_name, len, copts, log, ',').
- fsopt->new_dev_syntax = false.

REQ-7: `canonicalize_path(path)`:
- Collapse adjacent slashes: "//a///b//" → "/a/b".
- Drop trailing slash unless only character: "///" → "/".

REQ-8: `ceph_parse_mount_param(fc, param)`:
- /* Dispatch to libceph first */
- ret = ceph_parse_param(param, copts, log).
- if ret != -ENOPARAM: return ret.
- /* Else ceph-fs-specific */
- token = fs_parse(fc, ceph_mount_parameters, param, &result).
- switch token: Opt_wsize/Opt_rsize: range-check [PAGE_SIZE, CEPH_MAX_*]; ALIGN to PAGE_SIZE.
- Opt_recover_session: clean ⟹ flags |= CLEANRECOVER.
- Opt_fscache: requires CONFIG_CEPH_FSCACHE.
- Opt_acl: requires CONFIG_CEPH_FS_POSIX_ACL; sets/clears SB_POSIXACL.
- Opt_mds_namespace: if non-matching with prior, invalfc("Mismatching mds_namespace").
- Opt_source: invalfc if fc->source already set (multi-source forbidden).
- Opt_test_dummy_encryption: requires CONFIG_FS_ENCRYPTION; -EEXIST ⟹ conflict.

REQ-9: `create_fs_client(fsopt, opt)`:
- fsc = kzalloc.
- fsc->client = ceph_create_client(opt, fsc).
- fsc->client->extra_mon_dispatch = extra_mon_dispatch.
- ceph_set_opt(fsc->client, ABORT_ON_FULL).
- if !fsopt->mds_namespace: ceph_monc_want_map(monc, CEPH_SUB_MDSMAP, 0, true).
- else: ceph_monc_want_map(monc, CEPH_SUB_FSMAP, 0, false).
- mount_options = fsopt; mount_state = CEPH_MOUNT_MOUNTING; filp_gen = 1.
- inode_wq = alloc_workqueue("ceph-inode", WQ_UNBOUND, 0).
- cap_wq = alloc_workqueue("ceph-cap", WQ_PERCPU, 1).
- hash_init(async_unlink_conflict).
- list_add_tail(&fsc->metric_wakeup, &ceph_fsc_list).
- On failure: ceph_destroy_options(opt); destroy_mount_options(fsopt).

REQ-10: `extra_mon_dispatch(client, msg)`:
- CEPH_MSG_MDS_MAP: ceph_mdsc_handle_mdsmap(fsc->mdsc, msg).
- CEPH_MSG_FS_MAP_USER: ceph_mdsc_handle_fsmap(fsc->mdsc, msg).
- default: return -1 (not handled).

REQ-11: `ceph_real_mount(fsc, fc)`:
- started = jiffies.
- mutex_lock(&fsc->client->mount_mutex).
- if !fsc->sb->s_root:
  - path = mount_options->server_path ? server_path + 1 : "".
  - __ceph_open_session(fsc->client) — mon-authenticates, gets monmap, gets initial maps.
  - if FSCACHE flag: ceph_fscache_register_fs(fsc, fc).
  - ceph_apply_test_dummy_encryption(sb, fc, mount_options).
  - ceph_fs_debugfs_init(fsc).
  - root = open_root_dentry(fsc, path, started).
  - fsc->sb->s_root = dget(root).
- else: root = dget(fsc->sb->s_root) (shared sb path).
- fsc->mount_state = CEPH_MOUNT_MOUNTED.
- mutex_unlock; return root.

REQ-12: `open_root_dentry(fsc, path, started)`:
- req = ceph_mdsc_create_request(mdsc, CEPH_MDS_OP_GETATTR, USE_ANY_MDS).
- req->r_path1 = kstrdup(path).
- req->r_ino1 = {ino = CEPH_INO_ROOT, snap = CEPH_NOSNAP}.
- req->r_started = started.
- req->r_timeout = client->options->mount_timeout.
- req->r_args.getattr.mask = CEPH_STAT_CAP_INODE.
- req->r_num_caps = 2.
- err = ceph_mdsc_do_request(mdsc, NULL, req).
- if err == 0: inode = req->r_target_inode; root = d_make_root(inode).
- ceph_mdsc_put_request(req).

REQ-13: `ceph_set_super(s, fc)`:
- s->s_maxbytes = MAX_LFS_FILESIZE.
- s->s_xattr = ceph_xattr_handlers.
- fsc->sb = s; fsc->max_file_size = 1ULL << 40 (placeholder until mdsmap arrives).
- s->s_op = &ceph_super_ops.
- set_default_d_op(s, &ceph_dentry_ops).
- s->s_export_op = &ceph_export_ops.
- s->s_time_gran = 1; s->s_time_min = 0; s->s_time_max = U32_MAX.
- s->s_flags |= SB_NODIRATIME | SB_NOATIME.
- s->s_magic = CEPH_SUPER_MAGIC.
- ceph_fscrypt_set_ops(s).
- set_anon_super_fc(s, fc).

REQ-14: `ceph_compare_super(sb, fc)` — predicate for sb-sharing:
- compare_mount_options(new fsopt, new opt, existing fsc) == 0.
- if CEPH_OPT_FSID set: fsids must match.
- fc->sb_flags == (sb->s_flags & ~SB_BORN).
- !fsc->blocklisted OR CEPH_MOUNT_OPT_CLEANRECOVER set.
- fsc->mount_state != CEPH_MOUNT_SHUTDOWN.

REQ-15: `ceph_get_tree(fc)`:
- if !fc->source: invalfc("No source").
- if new_dev_syntax ∧ !mon_addr: invalfc("No monitor address").
- fsc = create_fs_client(opts, copts).
- ceph_mdsc_init(fsc).
- if ceph_test_opt(client, NOSHARE): compare_super = NULL.
- fc->s_fs_info = fsc; sb = sget_fc(fc, compare_super, ceph_set_super).
- if ceph_sb_to_fs_client(sb) != fsc: existing client (destroy_fs_client(fsc)).
- else: ceph_setup_bdi(sb, fsc) (BDI name "ceph-%ld"; ra_pages = rasize >> PAGE_SHIFT; io_pages = rsize >> PAGE_SHIFT).
- res = ceph_real_mount(fsc, fc).
- on failure (no MDSes available): -EHOSTUNREACH; ceph_mdsc_close_sessions; deactivate_locked_super.

REQ-16: `ceph_kill_sb(s)`:
- ceph_mdsc_pre_umount(mdsc) — abort outstanding mds requests.
- flush_fs_workqueues(fsc) — drain inode_wq + cap_wq.
- sync_filesystem(s) — early flush (before stage bump) to drop further MDS msg-driven inode pins.
- if dirty_folios > 0: wait_event_killable_timeout(flush_end_wq, dirty_folios <= 0, mount_timeout).
- mdsc->stopping = CEPH_MDSC_STOPPING_FLUSHING.
- if stopping_blockers > 0: wait_for_completion_killable_timeout(stopping_waiter, mount_timeout).
- mdsc->stopping = CEPH_MDSC_STOPPING_FLUSHED.
- kill_anon_super(s).
- ceph_fs_debugfs_cleanup; ceph_fscache_unregister_fs.
- destroy_fs_client(fsc).

REQ-17: `ceph_force_reconnect(sb)`:
- fsc->mount_state = CEPH_MOUNT_RECOVER.
- __ceph_umount_begin(fsc) — abort osdc, force_umount mdsc, bump filp_gen.
- flush_workqueue(inode_wq).
- ceph_reset_client_addr(client) — resets blocklisted mon/osd connections.
- ceph_osdc_clear_abort_err.
- fsc->blocklisted = false; mount_state = MOUNTED.
- __ceph_do_getattr(d_inode(s->s_root), NULL, CEPH_STAT_CAP_INODE, true).

REQ-18: `ceph_statfs(dentry, buf)`:
- monc = &fsc->client->monc.
- data_pool = (num_data_pg_pools == 1) ? m_data_pg_pools[0] : CEPH_NOPOOL.
- ceph_monc_do_statfs(monc, data_pool, &st).
- buf->f_type = CEPH_SUPER_MAGIC.
- buf->f_frsize = 1 << CEPH_BLOCK_SHIFT; f_bsize = f_frsize.
- if NOQUOTADF set ∨ !ceph_quota_update_statfs: blocks/bfree/bavail from st.kb.
- buf->f_files = st.num_objects; f_ffree = -1; f_namelen = NAME_MAX.
- buf->f_fsid.val[0] = XOR-fold(monmap->fsid); val[1] = monc->fs_cluster_id.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mount_options_freed_on_error` | INVARIANT | per-create_fs_client error path: opts/copts/fsopt destroyed exactly once. |
| `pctx_freed_on_init_failure` | INVARIANT | per-init_fs_context nomem: pctx.opts/copts freed. |
| `compare_super_fsid_check` | INVARIANT | per-compare_super: fsid mismatch ⟹ return 0. |
| `wsize_rsize_in_range_and_aligned` | INVARIANT | per-parse: wsize/rsize ∈ [PAGE_SIZE, CEPH_MAX_*] ∧ PAGE_SIZE-aligned. |
| `server_path_canonical` | INVARIANT | per-canonicalize_path: no double-slash, no trailing slash (unless len==1). |
| `mds_namespace_exclusive_with_old_syntax` | INVARIANT | per-parse: new syntax sets new_dev_syntax=true; show_options conditioned. |
| `stopping_blocker_ref_balanced` | INVARIANT | per-inc/dec: stopping_blockers count returns to 0. |
| `mount_state_transition_legal` | INVARIANT | per-state: MOUNTING→MOUNTED→{SHUTDOWN, RECOVER}; no back-edges except RECOVER→MOUNTED. |

### Layer 2: TLA+

`fs/ceph/super.tla`:
- Per-init_fs_context + per-parse + per-get_tree + per-real_mount + per-kill_sb + per-force_reconnect.
- Properties:
  - `safety_no_sharing_when_shutdown` — per-compare_super: mount_state == SHUTDOWN excludes sharing.
  - `safety_no_sharing_when_blocklisted` — per-compare_super: blocklisted ∧ !CLEANRECOVER excludes sharing.
  - `safety_blocking_drain_before_kill` — per-kill_sb: dirty_folios == 0 ∧ stopping_blockers == 0 before kill_anon_super (or timeout).
  - `safety_open_root_under_mount_mutex` — per-real_mount: open_root_dentry called with mount_mutex held.
  - `liveness_get_tree_terminates_or_errors` — per-mount: eventual success ∨ defined error.
  - `liveness_force_reconnect_returns_to_mounted` — per-recover: state == MOUNTED after successful reconnect.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `CephSuper::parse_source` post: server_path canonical OR mon_addr set | `CephSuper::parse_source` |
| `CephSuper::parse_new_source` post: new_dev_syntax=true ∧ mds_namespace set | `CephSuper::parse_new_source` |
| `CephFsClient::create` post: workqueues alive ∧ on ceph_fsc_list ∨ all resources freed | `CephFsClient::create` |
| `CephSuper::set_super` post: s_op/s_xattr/s_magic/s_export_op set ∧ SB_NODIRATIME|SB_NOATIME | `CephSuper::set_super` |
| `CephSuper::compare_super` post: all 5 predicates evaluated; mismatch ⟹ 0 | `CephSuper::compare_super` |
| `CephSuper::kill_sb` post: client destroyed ∧ workqueues drained ∧ fscache unregistered | `CephSuper::kill_sb` |
| `CephSuper::real_mount` post: mount_state == Mounted ∧ s_root pinned | `CephSuper::real_mount` |
| `CephSuper::force_reconnect` post: blocklisted == false ∧ mount_state == Mounted | `CephSuper::force_reconnect` |

### Layer 4: Verus/Creusot functional

`Per-fs_context init → per-parse_param tokens → per-get_tree (create_fs_client + mdsc_init + sget_fc + real_mount) → per-real_mount (open_session + open_root_dentry) → per-kill_sb teardown` semantic equivalence: per-Documentation/filesystems/ceph.rst.

### hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

Ceph fs-client reinforcement:

- **Per-mount_options bounded** — defense against per-injection (wsize/rsize range-checked, snapdirname ≤ NAME_MAX, server_path canonicalised).
- **Per-libceph error consumed exactly once** — defense against per-double-free (create_fs_client documents "consumes fsopt and opt").
- **Per-mount_mutex serialisation** — defense against per-concurrent-mount on same client.
- **Per-stopping_blockers + flush_end_wq drain on kill_sb** — defense against per-UAF (inode-pins via in-flight MDS messages).
- **Per-blocklisted re-mount requires CLEANRECOVER** — defense against per-stale-state revival.
- **Per-MOUNTWAIT bounded by mount_timeout** — defense against per-unbounded-hang on dead MDS.
- **Per-extra_mon_dispatch nulled on kill_sb** — defense against per-stray-msg after destroy.
- **Per-fscache_uniq strict-string** — defense against per-cache-namespace collision.
- **Per-new_dev_syntax requires mon_addr** — defense against per-misparsed-mount.
- **Per-fsid match on sb-share** — defense against per-cross-cluster collision.
- **Per-filp_gen bump on force_umount** — defense against per-stale-fd post-recover.
- **Per-test_dummy_encryption immutable on remount** — defense against per-key-substitution.
- **Per-CAP_SYS_ADMIN for mount** — inherited from VFS.

