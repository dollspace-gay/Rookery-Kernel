# Tier-3: fs/afs/super.c — Andrew File System (kAFS) client superblock + mount

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/00-overview.md
upstream-paths:
  - fs/afs/super.c (~770 lines)
  - fs/afs/internal.h (afs_super_info, afs_fs_context, afs_net, afs_vnode)
  - fs/afs/cell.c (afs_lookup_cell, afs_find_cell, alias detection)
  - fs/afs/volume.c (afs_create_volume, afs_activate_volume, afs_deactivate_volume)
  - fs/afs/inode.c (afs_root_iget, afs_evict_inode)
  - fs/afs/dynroot.c (afs_dynroot_iget_root)
  - fs/afs/vlclient.c (VL.GetEntryByNameU RPC)
  - net/rxrpc/* (Rx/UDP RPC transport)
-->

## Summary

`fs/afs/super.c` is the **VFS-facing entry point** for the kAFS in-kernel Andrew File System client. It owns: (a) the `afs_super_info` per-superblock record (cell, volume, net_ns, flock_mode, dyn_root), (b) the `afs_fs_context` per-mount parse state (cell, volume, key, type, force, autocell, dyn_root, no_cell, flock_mode), (c) the source-name parser for AFS device strings (`%cell:volume.suffix` / `#cell:volume.suffix` / `none`), (d) the `afs_validate_fc` flow that requests a kerberos/RxRPC key, detects cell aliases, and creates the `afs_volume` via VL-server lookup, (e) the `fill_super` flow that allocates the AFS root vnode (afs_root_iget triggers a FetchStatus RPC) or the dynroot, and (f) `afs_kill_super` teardown. Per-superblock sharing is keyed on (net_ns, volume->vid, cell, !dyn_root). Per-AFS: kAFS is a NETWORK fs using the Rx RPC protocol over UDP with Kerberos auth; its mount idiom differs sharply from POSIX-local fs.

This Tier-3 covers `fs/afs/super.c` (~770 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct afs_super_info` | per-sb state (net_ns, cell, volume, flock_mode, dyn_root) | `AfsSuperInfo` |
| `struct afs_fs_context` | per-fc parse state | `AfsFsContext` |
| `struct afs_volume` | per-volume runtime (vid, name, type, sb, servers, locks) | `AfsVolume` |
| `struct afs_cell` | per-cell record (name, vlservers, alias_of, flags) | `AfsCell` |
| `struct afs_vnode` | per-AFS-inode (fid, status, volume, lock_state, cb_*) | `AfsVnode` |
| `struct afs_net` | per-net-namespace AFS state | `AfsNet` |
| `afs_fs_type` | per-file_system_type ("afs") | `AFS_FS_TYPE` |
| `afs_super_ops` | per-super_operations vtable | `AFS_SUPER_OPS` |
| `afs_context_ops` | per-fs_context_operations vtable | `AFS_CONTEXT_OPS` |
| `afs_fs_parameters[]` | per-fs_parameter_spec table (autocell, dyn, flock, source) | `AFS_FS_PARAMETERS` |
| `afs_param_flock[]` | per-flock enum (local, openafs, strict, write) | `AFS_PARAM_FLOCK` |
| `afs_init_fs_context()` | per-fc init: alloc ctx, set default cell | `AfsSuper::init_fs_context` |
| `afs_parse_param()` | per-token dispatch | `AfsSuper::parse_param` |
| `afs_parse_source()` | per-`%cell:volume.suffix` parser | `AfsSuper::parse_source` |
| `afs_validate_fc()` | per-fc: request key, alias-detect, create volume | `AfsSuper::validate_fc` |
| `afs_get_tree()` | per-fc top-level mount driver | `AfsSuper::get_tree` |
| `afs_test_super()` | per-sb share predicate (regular) | `AfsSuper::test_super` |
| `afs_dynroot_test_super()` | per-sb share predicate (dynroot) | `AfsSuper::dynroot_test_super` |
| `afs_set_super()` | per-sb init (anon-super) | `AfsSuper::set_super` |
| `afs_fill_super()` | per-sb fill (s_op, root inode, dentry) | `AfsSuper::fill_super` |
| `afs_alloc_sbi()` | per-fc → super_info | `AfsSuper::alloc_sbi` |
| `afs_destroy_sbi()` | per-sb teardown | `AfsSuper::destroy_sbi` |
| `afs_kill_super()` | per-`kill_sb`: sb-deactivate + volume-deactivate | `AfsSuper::kill_super` |
| `afs_free_fc()` | per-fc dispose | `AfsSuper::free_fc` |
| `afs_show_devname()` | per-/proc/mounts device | `AfsSuper::show_devname` |
| `afs_show_options()` | per-/proc/mounts options | `AfsSuper::show_options` |
| `afs_statfs()` | per-`statfs()`: VL/YFS GetVolumeStatus | `AfsSuper::statfs` |
| `afs_alloc_inode()` | per-inode alloc from kmem_cache | `AfsSuper::alloc_inode` |
| `afs_free_inode()` | per-inode free | `AfsSuper::free_inode` |
| `afs_destroy_inode()` | per-inode destroy (counter decrement) | `AfsSuper::destroy_inode` |
| `afs_i_init_once()` | per-slab-init once | `AfsSuper::inode_init_once` |
| `afs_fs_init()` | per-module init: kmem_cache + register_filesystem | `AfsSuper::fs_init` |
| `afs_fs_exit()` | per-module exit: unreg + drain + kmem_cache_destroy | `AfsSuper::fs_exit` |
| `afs_get_volume_status_success()` | per-statfs RPC success callback | `AfsSuper::get_volume_status_success` |
| `afs_get_volume_status_operation` | per-statfs operation ops | `AFS_GET_VOLUME_STATUS_OP` |

## Compatibility contract

REQ-1: `struct afs_super_info` (per-superblock):
- net_ns: per-mount net namespace (get_net'd, put_net on destroy).
- cell: `struct afs_cell *` (afs_use_cell'd, afs_unuse_cell on destroy).
- volume: `struct afs_volume *` (afs_get_volume'd, afs_put_volume on destroy).
- flock_mode: afs_flock_mode_{unset,local,openafs,strict,write}.
- dyn_root: bool — special "dyn" superblock acting as cell-aware dynamic mount-point root.

REQ-2: `struct afs_fs_context` (per-fc-private, transient):
- net: `struct afs_net *` (resolved from fc->net_ns).
- cell: candidate `afs_cell *` (used+unused over fc lifetime).
- volume: candidate `afs_volume *` (alloc'd by validate, transferred to super_info on success).
- key: kernel keyring `struct key *` (kerberos token, key_put on free_fc).
- volname: pointer into source string (start of volume name).
- volnamesz: derived volume-name byte length.
- type: AFSVL_RWVOL | AFSVL_ROVOL | AFSVL_BACKVOL.
- force: bool — explicit suffix locks volume-type choice.
- autocell: bool — mount-option `autocell` (lazy cell creation under dyn root).
- dyn_root: bool — mount-option `dyn`.
- no_cell: bool — source=="none" with `-o dyn`.
- flock_mode: enum (initialised from `-o flock=` if any).

REQ-3: `afs_fs_parameters[]` (per-fs_parameter_spec):
- `autocell` flag — Opt_autocell.
- `dyn` flag — Opt_dyn.
- `flock` enum (local/openafs/strict/write) — Opt_flock.
- `source` string — Opt_source.

REQ-4: `afs_parse_source(fc, param)` — source name syntax:
- "%[cell:]volume[.]" → R/W volume.
- "#[cell:]volume[.]" → R/O or R/W volume (parent-dependent).
- "%[cell:]volume.readonly" → R/O (forced).
- "#[cell:]volume.readonly" → R/O (forced).
- "%[cell:]volume.backup" → backup volume (forced).
- "#[cell:]volume.backup" → backup volume (forced).
- "none" (only with -o dyn): no_cell = true.
- Multi-source: invalf "Multiple sources not supported".
- Unparsable: -EINVAL (kAFS: unparsable volume name).
- If '%': type = AFSVL_RWVOL, force = true.
- Split on ':' for cell-name prefix.
- Split on '.' for type-suffix.
- If cellname present: afs_lookup_cell(net, name, sz, NULL, AFS_LOOKUP_CELL_DIRECT_MOUNT, …) and afs_unuse_cell(prev) + afs_see_cell(new).
- fc->source = param->string (param's string ownership transferred).

REQ-5: `afs_parse_param(fc, param)`:
- opt = fs_parse(fc, afs_fs_parameters, param, &result).
- Opt_source: afs_parse_source.
- Opt_autocell: ctx.autocell = true.
- Opt_dyn: ctx.dyn_root = true.
- Opt_flock: ctx.flock_mode = result.uint_32.
- default: -EINVAL.

REQ-6: `afs_validate_fc(fc)` — pre-get_tree gating:
- if !dyn_root:
  - if no_cell: pr_warn("Can only specify source 'none' with -o dyn"); -EINVAL.
  - if !cell: pr_warn("No cell specified"); -EDESTADDRREQ.
- reget_key:
  - key = afs_request_key(ctx.cell). On failure: PTR_ERR.
  - ctx.key = key.
  - if ctx.volume: afs_put_volume(volume); volume = NULL.
  - if AFS_CELL_FL_CHECK_ALIAS on cell: afs_cell_detect_alias(cell, key).
    - on alias (ret==1): key_put(key); switch cell to alias_of via afs_use_cell + afs_unuse_cell; goto reget_key.
  - volume = afs_create_volume(ctx). On failure: PTR_ERR.
  - ctx.volume = volume.
  - if volume.type != AFSVL_RWVOL: ctx.flock_mode = afs_flock_mode_local; fc->sb_flags |= SB_RDONLY.

REQ-7: `afs_get_tree(fc)`:
- ret = afs_validate_fc(fc). On error: out.
- as = afs_alloc_sbi(fc). On nomem: -ENOMEM.
- fc->s_fs_info = as.
- sb = sget_fc(fc, as.dyn_root ? afs_dynroot_test_super : afs_test_super, afs_set_super).
- if !sb.s_root: afs_fill_super(sb, ctx); sb.s_flags |= SB_ACTIVE.
- else: ASSERTCMP(s_flags & SB_ACTIVE).
- fc->root = dget(sb.s_root).
- trace_afs_get_tree(cell, volume).

REQ-8: `afs_test_super(sb, fc)` — sharing predicate (regular):
- as.net_ns == fc.net_ns.
- as.volume != NULL.
- as.volume.vid == fc.fs_private.volume.vid.
- as.cell == ctx.cell.
- !as.dyn_root.

REQ-9: `afs_dynroot_test_super(sb, fc)` — sharing predicate (dynroot):
- as.net_ns == fc.net_ns ∧ as.dyn_root.

REQ-10: `afs_set_super(sb, fc)`:
- set_anon_super(sb, NULL) — no backing device.

REQ-11: `afs_fill_super(sb, ctx)`:
- sb.s_blocksize = PAGE_SIZE; s_blocksize_bits = PAGE_SHIFT.
- sb.s_maxbytes = MAX_LFS_FILESIZE.
- sb.s_magic = AFS_FS_MAGIC.
- sb.s_op = &afs_super_ops.
- if !dyn_root: sb.s_xattr = afs_xattr_handlers.
- super_setup_bdi(sb).
- if dyn_root: inode = afs_dynroot_iget_root(sb).
- else: snprintf(sb.s_id, "%llu", volume.vid); afs_activate_volume(volume); inode = afs_root_iget(sb, ctx.key) — issues VL/FS FetchStatus RPC.
- sb.s_root = d_make_root(inode).
- if dyn_root: set_default_d_op(sb, &afs_dynroot_dentry_operations).
- else: set_default_d_op(sb, &afs_fs_dentry_operations); rcu_assign_pointer(volume.sb, sb).

REQ-12: `afs_alloc_sbi(fc)`:
- as = kzalloc<afs_super_info>.
- as.net_ns = get_net(fc.net_ns).
- as.flock_mode = ctx.flock_mode.
- if ctx.dyn_root: as.dyn_root = true.
- else: as.cell = afs_use_cell(ctx.cell, trace_use_sbi); as.volume = afs_get_volume(ctx.volume, trace_get_alloc_sbi).

REQ-13: `afs_destroy_sbi(as)`:
- afs_put_volume(as.volume, trace_put_destroy_sbi).
- afs_unuse_cell(as.cell, trace_unuse_sbi).
- put_net(as.net_ns).
- kfree(as).

REQ-14: `afs_kill_super(sb)`:
- if as.volume: rcu_assign_pointer(as.volume.sb, NULL) — clear before deactivate to stop callback ilookup5 races.
- kill_anon_super(sb).
- if as.volume: afs_deactivate_volume(as.volume).
- afs_destroy_sbi(as).

REQ-15: `afs_free_fc(fc)`:
- afs_destroy_sbi(fc.s_fs_info).
- afs_put_volume(ctx.volume, trace_put_free_fc).
- afs_unuse_cell(ctx.cell, trace_unuse_fc).
- key_put(ctx.key).
- kfree(ctx).

REQ-16: `afs_init_fs_context(fc)`:
- ctx = kzalloc<afs_fs_context>.
- ctx.type = AFSVL_ROVOL (default before suffix-override).
- ctx.net = afs_net(fc.net_ns).
- cell = afs_find_cell(net, NULL, 0, trace_use_fc) — workstation default; ERR_PTR ⟹ NULL.
- ctx.cell = cell.
- fc.fs_private = ctx; fc.ops = &afs_context_ops.

REQ-17: `afs_show_devname(m, root)`:
- if dyn_root: "none".
- else: prefix = '%' (RW) or '#' (RO/BACK); suffix = "" / ".readonly" / ".backup".
- "{pref}{cell.name}:{volume.name}{suffix}".

REQ-18: `afs_show_options(m, root)`:
- if dyn_root: ",dyn".
- if flock_mode != unset: ",flock={local|openafs|strict|write}".

REQ-19: `afs_statfs(dentry, buf)`:
- buf.f_type = s_magic; f_bsize = AFS_BLOCK_SIZE; f_namelen = AFSNAMEMAX - 1.
- if dyn_root: f_blocks=1, f_bavail=f_bfree=0.
- else: op = afs_alloc_operation(NULL, volume); afs_op_set_vnode(op, 0, vnode); op.nr_files=1; op.volstatus.buf=buf; op.ops=&afs_get_volume_status_operation; afs_do_sync_operation(op).
- success callback: if max_quota == 0: f_blocks = part_max_blocks; else: f_blocks = max_quota; f_bavail = f_bfree = max(0, f_blocks - blocks_in_use).

## Acceptance Criteria

- [ ] AC-1: `mount -t afs %example.com:user.home /mnt/afs` parses cell="example.com", volume="user.home", type=AFSVL_RWVOL, force=true.
- [ ] AC-2: `mount -t afs '#example.com:user.home.readonly' /mnt/afs` parses type=AFSVL_ROVOL, sb_flags |= SB_RDONLY, flock_mode=local.
- [ ] AC-3: `mount -t afs none /mnt/afs -o dyn` parses no_cell=true, dyn_root=true; succeeds with dynroot superblock.
- [ ] AC-4: `mount -t afs none /mnt/afs` (no -o dyn) ⟹ -EINVAL ("source 'none' with -o dyn").
- [ ] AC-5: `mount -t afs %volume /mnt/afs` (no cell, no default cell) ⟹ -EDESTADDRREQ ("No cell specified").
- [ ] AC-6: Cell with AFS_CELL_FL_CHECK_ALIAS set: afs_cell_detect_alias triggers, key re-requested, ctx.cell switched to alias_of.
- [ ] AC-7: afs_get_tree on existing-matching sb: sget_fc returns existing sb (s_flags & SB_ACTIVE); fc->root pinned via dget.
- [ ] AC-8: afs_test_super rejects sharing when net_ns differs, volume vid differs, cell differs, or dyn_root differs.
- [ ] AC-9: afs_fill_super for regular mount: afs_activate_volume called; volume.sb published via rcu_assign_pointer.
- [ ] AC-10: afs_kill_super: volume.sb cleared before kill_anon_super; afs_deactivate_volume called after.
- [ ] AC-11: afs_statfs on dynroot: f_blocks=1, f_bavail=0 (no RPC issued).
- [ ] AC-12: afs_statfs on volume: VL.GetVolumeStatus RPC issued; max_quota==0 path uses part_max_blocks.
- [ ] AC-13: `-o flock=strict` parsed; show_options prints ",flock=strict".
- [ ] AC-14: afs_free_fc: ctx.key key_put'd, ctx.volume put, ctx.cell unused.
- [ ] AC-15: afs_alloc_inode increments afs_count_active_inodes; afs_destroy_inode decrements; afs_fs_exit BUG()s if count != 0.

## Architecture

```
struct AfsSuperInfo {
  net_ns: *Net,                            // get_net'd
  cell: *AfsCell,                          // afs_use_cell'd
  volume: *AfsVolume,                      // afs_get_volume'd
  flock_mode: AfsFlockMode,                // unset|local|openafs|strict|write
  dyn_root: bool,
}

struct AfsFsContext {
  net: *AfsNet,                            // afs_net(net_ns)
  cell: Option<*AfsCell>,                  // candidate
  volume: Option<*AfsVolume>,              // candidate
  key: Option<*Key>,                       // RxRPC/krb5 token
  volname: *const u8,
  volnamesz: usize,
  type: AfsVolType,                        // RWVOL|ROVOL|BACKVOL
  force: bool,                             // suffix forced type
  autocell: bool,
  dyn_root: bool,
  no_cell: bool,                           // source == "none"
  flock_mode: AfsFlockMode,
}

enum AfsFlockMode {
  Unset,
  Local,
  OpenAfs,
  Strict,
  Write,
}

enum AfsVolType {
  RwVol = AFSVL_RWVOL,
  RoVol = AFSVL_ROVOL,
  BackVol = AFSVL_BACKVOL,
}
```

`AfsSuper::init_fs_context(fc) -> Result<()>`:
1. ctx = kzalloc<AfsFsContext>.
2. ctx.type = AfsVolType::RoVol (default; suffix may override).
3. ctx.net = afs_net(fc.net_ns).
4. cell = afs_find_cell(ctx.net, NULL, 0, trace_use_fc) /* workstation cell */.
5. if cell is IS_ERR: cell = None.
6. ctx.cell = cell.
7. fc.fs_private = ctx; fc.ops = &AFS_CONTEXT_OPS.

`AfsSuper::parse_param(fc, param) -> Result<()>`:
1. opt = fs_parse(fc, AFS_FS_PARAMETERS, param, &result).
2. if opt < 0: return opt.
3. match opt: { Opt_source: parse_source, Opt_autocell: ctx.autocell=true, Opt_dyn: ctx.dyn_root=true, Opt_flock: ctx.flock_mode=result.uint_32, _: -EINVAL }.

`AfsSuper::parse_source(fc, param) -> Result<()>`:
1. if fc.source.is_some(): invalf("Multiple sources not supported").
2. name = param.string. if !name: -EINVAL "no volume name specified".
3. if (name[0] != '%' ∧ name[0] != '#') ∨ !name[1]:
   - if name == "none": ctx.no_cell = true; return Ok.
   - else: -EINVAL "unparsable volume name".
4. if name[0] == '%': ctx.type = RwVol; ctx.force = true.
5. name = &name[1..].
6. /* Split cell:volume on ':' */
7. ctx.volname = strchr(name, ':') ?? then advance past it, with cellname = name and cellnamesz computed.
8. else: ctx.volname = name; cellname = None.
9. /* Type-suffix detection */
10. suffix = strrchr(ctx.volname, '.').
11. if suffix == ".readonly": ctx.type = RoVol; ctx.force = true.
12. else if suffix == ".backup": ctx.type = BackVol; ctx.force = true.
13. else if suffix[1] == 0: /* trailing dot only */.
14. else: suffix = None.
15. ctx.volnamesz = suffix ? suffix - volname : strlen(volname).
16. if cellname.is_some():
    - cell = afs_lookup_cell(ctx.net, cellname, cellnamesz, NULL, AFS_LOOKUP_CELL_DIRECT_MOUNT, trace_use_lookup_mount).
    - afs_unuse_cell(ctx.cell, trace_unuse_parse).
    - afs_see_cell(cell, trace_see_source).
    - ctx.cell = cell.
17. fc.source = param.string; param.string = NULL.

`AfsSuper::validate_fc(fc) -> Result<()>`:
1. if !ctx.dyn_root:
   - if ctx.no_cell: pr_warn(...); -EINVAL.
   - if !ctx.cell: pr_warn("No cell specified"); -EDESTADDRREQ.
2. loop /* "reget_key" */:
   - key = afs_request_key(ctx.cell). On err: PTR_ERR.
   - ctx.key = key.
   - if ctx.volume.is_some(): afs_put_volume(ctx.volume); ctx.volume = None.
   - if test_bit(AFS_CELL_FL_CHECK_ALIAS, &ctx.cell.flags):
     - ret = afs_cell_detect_alias(ctx.cell, key).
     - if ret < 0: return ret.
     - if ret == 1: /* alias switch */
       - key_put(ctx.key); ctx.key = None.
       - cell = afs_use_cell(ctx.cell.alias_of, trace_use_fc_alias).
       - afs_unuse_cell(ctx.cell, trace_unuse_fc).
       - ctx.cell = cell.
       - continue loop.
   - break.
3. volume = afs_create_volume(ctx). On err: PTR_ERR.
4. ctx.volume = Some(volume).
5. if volume.type != AfsVolType::RwVol:
   - ctx.flock_mode = AfsFlockMode::Local.
   - fc.sb_flags |= SB_RDONLY.

`AfsSuper::get_tree(fc) -> Result<()>`:
1. validate_fc(fc).
2. as = alloc_sbi(fc).
3. fc.s_fs_info = as.
4. sb = sget_fc(fc, as.dyn_root ? &dynroot_test_super : &test_super, &set_super).
5. if !sb.s_root: fill_super(sb, ctx); sb.s_flags |= SB_ACTIVE.
6. else: ASSERTCMP(sb.s_flags, has SB_ACTIVE).
7. fc.root = dget(sb.s_root).
8. trace_afs_get_tree(as.cell, as.volume).

`AfsSuper::fill_super(sb, ctx) -> Result<()>`:
1. sb.s_blocksize = PAGE_SIZE; s_blocksize_bits = PAGE_SHIFT.
2. sb.s_maxbytes = MAX_LFS_FILESIZE.
3. sb.s_magic = AFS_FS_MAGIC.
4. sb.s_op = &AFS_SUPER_OPS.
5. if !as.dyn_root: sb.s_xattr = AFS_XATTR_HANDLERS.
6. super_setup_bdi(sb).
7. inode = if as.dyn_root { afs_dynroot_iget_root(sb) } else {
   - snprintf(sb.s_id, "%llu", as.volume.vid).
   - afs_activate_volume(as.volume).
   - afs_root_iget(sb, ctx.key) /* issues FetchStatus RPC */.
   }.
8. sb.s_root = d_make_root(inode).
9. if as.dyn_root: set_default_d_op(sb, &AFS_DYNROOT_DENTRY_OPS).
10. else: set_default_d_op(sb, &AFS_FS_DENTRY_OPS); rcu_assign_pointer(as.volume.sb, sb).

`AfsSuper::alloc_sbi(fc) -> Option<*AfsSuperInfo>`:
1. as = kzalloc<AfsSuperInfo>.
2. as.net_ns = get_net(fc.net_ns).
3. as.flock_mode = ctx.flock_mode.
4. if ctx.dyn_root: as.dyn_root = true.
5. else: as.cell = afs_use_cell(ctx.cell, trace_use_sbi); as.volume = afs_get_volume(ctx.volume, trace_get_alloc_sbi).

`AfsSuper::kill_super(sb)`:
1. /* Clear callback interests (which would otherwise do ilookup5) BEFORE deactivating the sb. */
2. if as.volume: rcu_assign_pointer(as.volume.sb, NULL).
3. kill_anon_super(sb).
4. if as.volume: afs_deactivate_volume(as.volume).
5. afs_destroy_sbi(as).

`AfsSuper::statfs(dentry, buf) -> Result<()>`:
1. buf.f_type = sb.s_magic; f_bsize = AFS_BLOCK_SIZE; f_namelen = AFSNAMEMAX - 1.
2. if as.dyn_root: buf.f_blocks=1; f_bavail=f_bfree=0; return Ok.
3. op = afs_alloc_operation(NULL, as.volume).
4. afs_op_set_vnode(op, 0, vnode).
5. op.nr_files = 1; op.volstatus.buf = buf; op.ops = &AFS_GET_VOLUME_STATUS_OP.
6. afs_do_sync_operation(op) /* VL.GetVolumeStatus or YFS variant */.
7. success: if vs.max_quota == 0: buf.f_blocks = vs.part_max_blocks else buf.f_blocks = vs.max_quota.
8. if buf.f_blocks > vs.blocks_in_use: buf.f_bavail = buf.f_bfree = buf.f_blocks - vs.blocks_in_use.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `source_must_have_prefix_or_be_none` | INVARIANT | per-parse_source: name[0] ∈ {'%', '#'} ∨ name=="none" (+ dyn_root). |
| `dyn_root_requires_no_cell_compat` | INVARIANT | per-validate_fc: no_cell ⟹ dyn_root. |
| `key_put_on_alias_switch` | INVARIANT | per-validate_fc: alias-switch frees previous key. |
| `volume_freed_on_revalidate` | INVARIANT | per-validate_fc reget_key: prior volume afs_put_volume'd before re-create. |
| `sbi_net_ns_balanced` | INVARIANT | per-alloc_sbi/destroy_sbi: get_net/put_net paired. |
| `volume_sb_published_under_rcu` | INVARIANT | per-fill_super non-dyn: rcu_assign_pointer(volume.sb, sb) after activate. |
| `kill_super_clears_volume_sb_first` | INVARIANT | per-kill_super: volume.sb=NULL BEFORE kill_anon_super (callback race-free). |
| `ro_volume_forces_rdonly` | INVARIANT | per-validate_fc: type != RWVOL ⟹ SB_RDONLY ∧ flock_mode = local. |
| `test_super_keys_balanced` | INVARIANT | per-test_super: (net_ns, vid, cell, !dyn_root) all participate. |

### Layer 2: TLA+

`fs/afs/super.tla`:
- Per-init_fs_context + per-parse + per-validate_fc + per-get_tree + per-fill_super + per-kill_super.
- Properties:
  - `safety_dynroot_iso_from_volume_super` — per-test_super: dyn_root sb shares only with dyn_root sb in same net_ns.
  - `safety_volume_super_isolated_per_vid_per_cell` — per-test_super: regular sb shares only on matching (volume.vid, cell).
  - `safety_alias_loop_bounded` — per-validate_fc: AFS_CELL_FL_CHECK_ALIAS resolved within bounded iterations.
  - `safety_kill_super_clears_callback_sink` — per-kill_super: volume.sb null BEFORE deactivate.
  - `liveness_validate_fc_terminates` — per-validate_fc: alias-chain resolves to non-alias OR error.
  - `liveness_get_tree_returns_dentry_or_error` — per-mount: eventual fc.root ∨ defined error.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `AfsSuper::parse_source` post: ctx.volname is null-terminated suffix of param.string (until '.' or end) | `AfsSuper::parse_source` |
| `AfsSuper::validate_fc` post: dyn_root ⟹ volume==None ∧ cell-may-be-None; !dyn_root ⟹ volume!=None ∧ key!=None | `AfsSuper::validate_fc` |
| `AfsSuper::alloc_sbi` post: net_ns refcount += 1 ∧ (dyn_root ⟹ cell==None ∧ volume==None) | `AfsSuper::alloc_sbi` |
| `AfsSuper::destroy_sbi` post: net_ns refcount -= 1 ∧ as freed | `AfsSuper::destroy_sbi` |
| `AfsSuper::fill_super` post: sb.s_root pinned ∧ (dyn_root ⟹ AFS_DYNROOT_DENTRY_OPS) ∨ (regular ⟹ volume.sb published) | `AfsSuper::fill_super` |
| `AfsSuper::kill_super` post: volume.sb cleared ∧ deactivated ∧ as destroyed | `AfsSuper::kill_super` |
| `AfsSuper::statfs` post: dyn_root branch issues no RPC ∧ regular branch returns from afs_do_sync_operation | `AfsSuper::statfs` |
| `AfsSuper::free_fc` post: all of {ctx.volume, ctx.cell, ctx.key, ctx} released | `AfsSuper::free_fc` |

### Layer 4: Verus/Creusot functional

`Per-fs_context init → per-parse_param + parse_source (cell:volume.suffix) → per-validate_fc (request_key + alias_detect + create_volume) → per-get_tree (sget_fc + fill_super + activate_volume + root_iget FetchStatus RPC) → per-kill_super teardown (clear volume.sb + deactivate)` semantic equivalence: per-Documentation/filesystems/afs.rst.

## Hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

kAFS super reinforcement:

- **Per-source-name strict syntax** — defense against per-injection (only '%', '#', or "none" with -o dyn accepted).
- **Per-volume RW vs RO suffix-locked** — defense against per-type-spoof (suffix forces type+force=true).
- **Per-cell alias resolved to canonical** — defense against per-mount-to-stale-alias.
- **Per-key request before volume-create** — defense against per-unauth-mount (kerberos enforced).
- **Per-key dropped on alias retry** — defense against per-key-leak under alias-switch.
- **Per-volume.sb publishing under rcu_assign_pointer** — defense against per-callback-vs-mount race.
- **Per-volume.sb cleared BEFORE kill_anon_super** — defense against per-UAF on callback-driven ilookup5.
- **Per-RO/back volume ⟹ SB_RDONLY ∧ flock=local** — defense against per-cross-volume write-leak.
- **Per-dyn_root isolated test_super** — defense against per-cross-mount-type sharing.
- **Per-net_ns refcount strict get/put** — defense against per-net-namespace UAF.
- **Per-afs_count_active_inodes BUG-on-exit-leak** — defense against per-module-unload inode-leak.
- **Per-source 'none' restricted to -o dyn** — defense against per-empty-source confusion.
- **Per-validate_fc fail-closed on no_cell-without-dyn** — defense against per-misconfigured-mount.
- **Per-RxRPC traffic confined to ctx.key context** — defense against per-credential-leak across mounts.

## Grsecurity/PaX-style Reinforcement

Beyond the upstream hardening above, Rookery layers the following grsec/PaX-style controls onto `fs/afs/super.c`:

- **PAX_USERCOPY** — bounds-checks every mount-option string (cell name, volume name, source-spec parser) so a malformed `-o source=` cannot drive a slab overrun inside `afs_parse_source`.
- **PAX_KERNEXEC** — keeps `afs_super_ops`, `afs_dynroot_inode_operations`, and the `rxrpc_kernel_ops` table in read-only memory so a network-side bug cannot rewrite `kill_sb`/`statfs`.
- **PAX_RANDKSTACK** — randomizes kernel-stack offset on each `mount(2)`/`statfs(2)` so an attacker cannot deterministically probe the RxRPC handshake path.
- **PAX_REFCOUNT** — wraps `afs_cell`, `afs_volume`, and `afs_net` reference counts so a malicious server cannot drive a wrap during a callback-flood.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `afs_super_info`, `afs_volume`, and `rxrpc_call` buffers so leftover Kerberos ticket material cannot be recovered from the freelist.
- **PAX_UDEREF** — enforces user/kernel pointer separation across the long mount-context parser and the `add_key`/`request_key` fallthroughs.
- **PAX_RAP / kCFI** — forward-edge CFI on `afs_super_ops` and `rxrpc_kernel_ops` vtables so a corrupted ops pointer cannot redirect callback-break handling to a userspace gadget.
- **GRKERNSEC_HIDESYM** — strips kernel pointers from `afs_debug` traces and cell-alias resolution printks so an unprivileged user cannot harvest kASLR offsets from AFS error messages.
- **GRKERNSEC_DMESG** — gates dmesg on `CAP_SYSLOG` so RxRPC negotiation errors do not leak pointer material to unprivileged users.
- **kAFS cell mount hard-gated on `CAP_SYS_ADMIN`** — `afs_get_tree` rejects unprivileged callers, blocking unprivileged-mount of a cell against an unauthenticated VLDB server even when user-namespaces relax other mount gates.
- **krb5/RxRPC key zero-on-destroy** — `afs_super_info.key` and the underlying `rxrpc_key_payload` are wiped via `memzero_explicit` on key revoke / mount teardown so a kernel-memory disclosure bug cannot recover Kerberos ticket material.
- **Cell-alias resolution audit** — `afs_alias_lookup` resolutions that change the canonical cell name are logged via `audit_log` so a DNS-poisoning attack that redirects a mount to a hostile cell is observable post-hoc.
- **GRKERNSEC_HIDESYM on RxRPC reject/abort paths** — RxRPC error printks sanitize embedded pointers and ticket material so a hostile server cannot extract kernel addresses or key bytes via crafted RxRPC errors.

Rationale: kAFS mounts trust a remote VLDB+fileserver pair with arbitrary filesystem metadata and rely on Kerberos tickets stored in-kernel; combined with the long-lived `afs_cell`/`afs_volume` lifecycle this is a high-value cred-leak target, so capability gating + explicit key zeroing + RAP on the RxRPC ops vtable are layered on top of the upstream callback-break / publish-under-RCU defenses.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/afs/cell.c cell management (afs_lookup_cell, afs_cell_detect_alias) — covered in `cell.md` Tier-3 if expanded.
- fs/afs/volume.c volume management (afs_create_volume, afs_activate_volume) — covered in `volume.md` Tier-3 if expanded.
- fs/afs/inode.c inode lifecycle (afs_root_iget, afs_evict_inode) — covered in `inode.md` Tier-3 if expanded.
- fs/afs/dynroot.c dynamic-mount root (afs_dynroot_iget_root) — covered in `dynroot.md` Tier-3 if expanded.
- fs/afs/vlclient.c VL-server RPCs — covered in `vlclient.md` Tier-3 if expanded.
- fs/afs/fsclient.c file-server RPCs — covered in `fsclient.md` Tier-3 if expanded.
- fs/afs/flock.c byte-range locking — covered in `flock.md` Tier-3 if expanded.
- net/rxrpc/* Rx-RPC transport — covered in `net/rxrpc/*.md` Tier-3 if expanded.
- Kerberos/krb5 token management — covered in `crypto/krb5/*.md` Tier-3 if expanded.
- Implementation code
