# Tier-3: fs/overlayfs/super.c — overlayfs union mount

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/00-overview.md
upstream-paths:
  - fs/overlayfs/super.c (~1622 lines)
  - fs/overlayfs/ovl_entry.h (struct ovl_fs, ovl_layer, ovl_inode, ovl_config)
  - fs/overlayfs/overlayfs.h (OVL_XATTR_*, enum ovl_xattr, ovl_xattr_table[])
  - fs/overlayfs/copy_up.c (ovl_copy_up, ovl_copy_up_data, struct ovl_copy_up_ctx)
  - fs/overlayfs/params.c (mount-option parsing, ovl_fs_context)
  - fs/overlayfs/export.c (ovl_export_operations / ovl_export_fid_operations)
  - Documentation/filesystems/overlayfs.rst
-->

## Summary

Overlayfs is a *union* (stacked) filesystem: a writable **upperdir** sits atop a stack of read-only **lowerdir** layers. Lookups walk the stack top-down (upper → lower[0] → lower[1] → …); writes are redirected to upperdir via **copy-up** on first modification. Per-`struct ovl_fs`: one per superblock, owning `numlayer` layers (layer[0] is upper, layer[1..] are lower), workdir (transient renames + copy-up temp), optional indexdir (nfs_export hardlinks + metacopy lookup keying), and the merged `ovl_config` (upperdir, workdir, lowerdirs[], redirect_mode, metacopy, index, nfs_export, xino, userxattr, fsync_mode, verity_mode, default_permissions, casefold). Per-`struct ovl_layer`: { vfsmount, trap-inode, ovl_sb*, idx, fsid, has_xwhiteouts }. Per-`struct ovl_inode`: VFS inode plus { upperdentry, ovl_entry (lowerstack), redirect, lowerdata_redirect or ovl_dir_cache, version, flags, lock }. Per-trusted xattr taxonomy (or user.* if `userxattr`): `OVL_XATTR_OPAQUE` (directory opaque — hides lower), `OVL_XATTR_REDIRECT` (renamed-from-original path), `OVL_XATTR_ORIGIN` (file handle to lower-origin), `OVL_XATTR_IMPURE` (dir has copied-up children, listdir must merge), `OVL_XATTR_NLINK`, `OVL_XATTR_UPPER` (index-dir entry kind), `OVL_XATTR_UUID`, `OVL_XATTR_METACOPY` (upper has metadata only, body lives lower), `OVL_XATTR_PROTATTR`, `OVL_XATTR_XWHITEOUT`. Per-copy-up policy: `ovl_copy_up_data` triggers on first write; `metacopy` defers data copy until first data-modifying access. Per-`redirect_dir`: directory rename across layers leaves `OVL_XATTR_REDIRECT` pointing to original lower path, preserving lookup semantics. Per-`nfs_export`: requires `index=on`; readdir uses indexed mode (hashed file handles for stateless re-lookup across server reboots). Critical for: container layer images (Docker, containerd, podman), live-CD/USB persistence, A/B-update systems, build-cache layering.

This Tier-3 covers `fs/overlayfs/super.c` (~1622 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ovl_fs` | per-sb context | `OvlFs` |
| `struct ovl_layer` | per-layer (upper or lower) | `OvlLayer` |
| `struct ovl_sb` | per-underlying-superblock annotation (pseudo_dev, bad_uuid, is_lower) | `OvlSb` |
| `struct ovl_config` | per-mount options snapshot | `OvlConfig` |
| `struct ovl_inode` | per-ovl-inode (upperdentry, oe, redirect, …) | `OvlInode` |
| `struct ovl_entry` | per-dentry/inode lowerstack `__lowerstack[]` | `OvlEntry` |
| `struct ovl_path` | per-stack entry (layer*, dentry) | `OvlPath` |
| `struct ovl_fs_context` | per-fs_context parse state | `OvlFsContext` |
| `ovl_fill_super()` | per-mount entry point | `OvlFs::fill_super` |
| `ovl_fill_super_creds()` | per-mount-with-creator-creds | `OvlFs::fill_super_creds` |
| `ovl_put_super()` | per-umount | `OvlFs::put_super` |
| `ovl_sync_fs()` | per-syncfs | `OvlFs::sync_fs` |
| `ovl_statfs()` | per-statfs (delegated to upper or top-lower) | `OvlFs::statfs` |
| `ovl_super_operations` | per-sb-ops vtable | shared |
| `ovl_alloc_inode` / `ovl_free_inode` / `ovl_destroy_inode` | per-inode alloc/free | `OvlInode::alloc` / `free` / `destroy` |
| `ovl_inode_init_once()` | per-slab-ctor | `OvlInode::init_once` |
| `ovl_get_upper()` | per-upperdir setup + inuse trap | `OvlFs::get_upper` |
| `ovl_get_workdir()` | per-workdir setup (capabilities, xattr probe, fallbacks) | `OvlFs::get_workdir` |
| `ovl_make_workdir()` | per-workdir creation + feature probe | `OvlFs::make_workdir` |
| `ovl_workdir_create()` | per-named-subdir creator | `OvlFs::workdir_create` |
| `ovl_get_indexdir()` | per-indexdir setup | `OvlFs::get_indexdir` |
| `ovl_get_layers()` | per-lower-layer stack assembly | `OvlFs::get_layers` |
| `ovl_get_fsid()` / `ovl_get_data_fsid()` | per-layer fsid + uuid bookkeeping | `OvlFs::get_fsid` / `get_data_fsid` |
| `ovl_lower_dir()` | per-lower vfs_path resolve + checks | `OvlFs::lower_dir` |
| `ovl_check_namelen()` | per-statfs namelen-min track | `OvlFs::check_namelen` |
| `ovl_check_layer()` / `ovl_check_overlapping_layers()` | per-layer integrity (no nested overlay, no loop) | `OvlFs::check_layer` / `check_overlapping_layers` |
| `ovl_setup_trap()` | per-layer dentry trap | `OvlFs::setup_trap` |
| `ovl_report_in_use()` | per-inuse-lock error | `OvlFs::report_in_use` |
| `ovl_check_rename_whiteout()` | per-renameat2-whiteout feature probe | `OvlFs::check_rename_whiteout` |
| `ovl_create_volatile_dirty()` | per-volatile-dirty marker | `OvlFs::create_volatile_dirty` |
| `ovl_lower_uuid_ok()` | per-uuid-ok-for-index/nfs_export | `OvlFs::lower_uuid_ok` |
| `ovl_set_encoding()` | per-casefold encoding propagation | `OvlFs::set_encoding` |
| `ovl_set_d_op()` | per-d_op vtable selection (revalidate or weak) | `OvlFs::set_d_op` |
| `ovl_dentry_revalidate()` / `_weak_revalidate()` / `_revalidate_common()` / `ovl_revalidate_real()` | per-dentry revalidation | `OvlFs::revalidate_*` |
| `ovl_workdir_ok()` | per-workdir-vs-upperdir validation | `OvlFs::workdir_ok` |
| `OVL_XATTR_OPAQUE / _REDIRECT / _ORIGIN / _IMPURE / _NLINK / _UPPER / _UUID / _METACOPY / _PROTATTR / _XWHITEOUT` | per-xattr namespace | shared uapi (trusted.overlay.* or user.overlay.*) |
| `ovl_export_operations` / `ovl_export_fid_operations` | per-export | shared |
| `ovl_xattr_handlers()` | per-xattr-handler vtable | `OvlFs::xattr_handlers` |
| `ovl_copy_up_data()` (copy_up.c) | per-data-copy-up | `OvlCopyUp::data` |
| `ovl_do_copy_up()` (copy_up.c) | per-copy-up dispatcher (workdir vs tmpfile) | `OvlCopyUp::do_copy_up` |
| `struct ovl_copy_up_ctx` | per-copy-up context | `OvlCopyUpCtx` |
| `OVL_WORKDIR_NAME` ("work") / `OVL_INDEXDIR_NAME` ("index") | per-name | shared |
| `OVERLAYFS_SUPER_MAGIC` | per-magic | shared |
| `ovl_get_root()` | per-root-dentry build | `OvlFs::get_root` |
| `ovl_get_lowerstack()` | per-root-lowerstack | `OvlFs::get_lowerstack` |

## Compatibility contract

REQ-1: struct ovl_config:
- upperdir: per-string upper layer path (or NULL ⟹ read-only).
- workdir: per-string work dir path (must share fs with upperdir).
- lowerdirs[]: per-string array; lowerdirs[0] holds raw colon-joined string for show_options; lowerdirs[1..numlayer] are per-layer paths.
- default_permissions: per-flag enforce upper-derived permission checks.
- redirect_mode: OVL_REDIRECT_OFF / _FOLLOW / _NOFOLLOW / _ON.
- verity_mode: OVL_VERITY_OFF / _ON / _REQUIRE.
- index: bool — indexdir present.
- uuid: OVL_UUID_OFF / _NULL / _AUTO / _ON.
- nfs_export: bool — requires index=on.
- xino: OVL_XINO_OFF / _ON / _AUTO.
- metacopy: bool.
- userxattr: bool (use user.overlay.* instead of trusted.overlay.*).
- fsync_mode: OVL_VOLATILE flag.

REQ-2: struct ovl_layer:
- mnt: vfsmount (cloned-private from underlying mount).
- trap: inode-trap for layer dir (used to detect mounting same path twice).
- fs: *ovl_sb (per-underlying-superblock annotation).
- idx: per-layer index in ofs.layers[] (0 = upper, 1..numlayer-1 = lower).
- fsid: per-unique-underlying-sb id (0 = upper).
- has_xwhiteouts: bool — layer has discovered XWHITEOUT xattrs.

REQ-3: struct ovl_sb:
- sb: *super_block of underlying fs.
- pseudo_dev: per-unique-pseudo-device assigned for inode-numbering.
- bad_uuid: bool — uuid conflicts with another layer (disables fsid).
- is_lower: bool — used as a lower (may also be upper for r/o-upper trick).

REQ-4: struct ovl_fs:
- numlayer, numfs, numdatalayer.
- layers: *ovl_layer (alloc length numlayer+1 in fill_super_creds).
- fs: *ovl_sb array (one per unique underlying sb).
- workbasedir: dentry of `workdir=` option.
- workdir: dentry of `work` or `index` subdir under workbasedir.
- namelen: per-min(statfs.f_namelen across layers).
- config: per-merged options snapshot.
- creator_cred: per-mount creator credentials (for VFS operations on lower as mounter, not caller).
- tmpfile, noxattr, nofh: per-feature flags discovered during mount.
- upperdir_locked, workdir_locked: per-inuse-lock state.
- workbasedir_trap, workdir_trap: per-trap inodes for overlay self-detection.
- xino_mode: -1 disabled, 0 same-fs, 1..32 number of high bits unused in lower inos for layer-fsid embed.
- last_ino: atomic_long for non-persistent inode-number allocation.
- whiteout: shared whiteout dentry (single char-device 0/0 file).
- no_shared_whiteout: bool fallback.
- whiteout_lock: mutex.
- errseq: per-volatile mount upperdir wb-err snapshot.
- casefold: bool — upper supports casefolding.

REQ-5: struct ovl_entry:
- __numlower: trailing-array length.
- __lowerstack[]: per-ovl_path entries (layer*, dentry) — top of stack first.

REQ-6: struct ovl_inode:
- union: cache (S_ISDIR) or lowerdata_redirect (S_ISREG).
- redirect: OVL_XATTR_REDIRECT path string.
- version: per-inode change-detection counter.
- flags: bitmap (OVL_UPPERDATA, OVL_IMPURE, OVL_WHITEOUTS, OVL_REDIRECT, OVL_METACOPY, OVL_UPPER_REDIRECT_LOCATED, OVL_XWHITEOUT, …).
- vfs_inode: embedded struct inode.
- __upperdentry: per-upper-side dentry (NULL until copy-up).
- oe: *ovl_entry (lowerstack).
- lock: per-copy-up serialization mutex.

REQ-7: ovl_fill_super(sb, fc):
- ofs = sb.s_fs_info (preallocated by params.c during fc_get_tree).
- if WARN_ON(fc.user_ns != current_user_ns()): -EIO.
- ovl_set_d_op(sb).
- if !ofs.creator_cred: ofs.creator_cred = prepare_creds().
- with_ovl_creds(sb) { err = ovl_fill_super_creds(fc, sb) }.
- on err: ovl_free_fs(ofs); sb.s_fs_info = NULL.
- return err.

REQ-8: ovl_fill_super_creds(fc, sb):
- ovl_fs_params_verify(ctx, &ofs.config).
- if ctx.nr == 0: pr_err "missing 'lowerdir'"; -EINVAL.
- layers = kzalloc_objs(ovl_layer, ctx.nr + 1).
- ofs.config.lowerdirs = kcalloc(ctx.nr + 1, sizeof(char*), GFP_KERNEL).
- ofs.config.lowerdirs[0] = ctx.lowerdir_all (raw colon-string).
- ofs.layers = layers; ofs.numlayer = 1 (layer 0 reserved for upper).
- sb.s_stack_depth = 0; sb.s_maxbytes = MAX_LFS_FILESIZE.
- atomic_long_set(&ofs.last_ino, 1).
- if config.xino != OFF: ofs.xino_mode = BITS_PER_LONG - 32 (or OFF on 32-bit kernel).
- sb.s_op = &ovl_super_operations.
- /* Upper */
- if config.upperdir:
  - if !config.workdir: pr_err "missing 'workdir'"; -EINVAL.
  - ovl_get_upper(sb, ofs, &layers[0], &ctx.upper) — opens upper, sets layers[0].mnt private, traps, MNT_INTERNAL.
  - if !ovl_should_sync(ofs): snapshot ofs.errseq from upper_sb.s_wb_err (volatile mount); if errseq_check non-zero: -EIO "Sync upperdir to clear state".
  - ovl_get_workdir(sb, ofs, &ctx.upper, &ctx.work) — opens/probes/setups workdir (calls ovl_make_workdir).
  - if !ofs.workdir: sb.s_flags |= SB_RDONLY (workdir failed → r/o).
  - sb.s_stack_depth = upper_sb.s_stack_depth.
  - sb.s_time_gran = upper_sb.s_time_gran.
- /* Lower stack */
- oe = ovl_get_lowerstack(sb, ctx, ofs, layers).
- if IS_ERR(oe): goto out_free_oe.
- if !ovl_upper_mnt(ofs): sb.s_flags |= SB_RDONLY.
- if ovl_has_fsid(ofs) ∧ ovl_upper_mnt(ofs): ovl_init_uuid_xattr(sb, ofs, &ctx.upper).
- /* Index */
- if !ovl_force_readonly(ofs) ∧ config.index:
  - ovl_get_indexdir(sb, ofs, oe, &ctx.upper).
  - if !ofs.workdir: sb.s_flags |= SB_RDONLY.
- ovl_check_overlapping_layers(sb, ofs) — refuse if any layer overlaps another layer's mount tree.
- /* Forced r/o post-conditions */
- if !ofs.workdir: ofs.config.index = false; if upper ∧ nfs_export: pr_warn fallback nfs_export=off.
- if config.metacopy ∧ config.nfs_export: pr_warn fallback nfs_export=off.
- /* Export op vtable */
- if config.nfs_export: sb.s_export_op = &ovl_export_operations.
- else if !ofs.nofh: sb.s_export_op = &ovl_export_fid_operations.
- cap_lower(creator_cred.cap_effective, CAP_SYS_RESOURCE).
- sb.s_magic = OVERLAYFS_SUPER_MAGIC.
- sb.s_xattr = ovl_xattr_handlers(ofs).
- sb.s_fs_info = ofs.
- sb.s_flags |= SB_POSIXACL (if CONFIG_FS_POSIX_ACL).
- sb.s_iflags |= SB_I_SKIP_SYNC | SB_I_NOUMASK | SB_I_EVM_HMAC_UNSUPPORTED.
- sb.s_root = ovl_get_root(sb, ctx.upper.dentry, oe).
- if !sb.s_root: -ENOMEM ⟹ ovl_free_entry(oe); err.
- return 0.

REQ-9: ovl_super_operations vtable:
- .alloc_inode = ovl_alloc_inode (kmem_cache_alloc from ovl_inode_cachep).
- .free_inode = ovl_free_inode (kfree(oi.redirect), kfree(oi.oe), mutex_destroy, kmem_cache_free).
- .destroy_inode = ovl_destroy_inode (dput(oi.__upperdentry); ovl_stack_put(lowerstack, numlower); ovl_dir_cache_free for dir; else kfree(lowerdata_redirect)).
- .drop_inode = inode_just_drop.
- .put_super = ovl_put_super (ovl_free_fs(ofs)).
- .sync_fs = ovl_sync_fs (delegate to upper_sb if wait ∧ ovl_should_sync).
- .statfs = ovl_statfs (vfs_statfs(real_path); patch f_namelen = ofs.namelen, f_type = OVERLAYFS_SUPER_MAGIC, f_fsid if has-fsid).
- .show_options = ovl_show_options (in params.c).

REQ-10: ovl_workdir_create(ofs, name, persist):
- dir = ofs.workbasedir.d_inode; mnt = ovl_upper_mnt(ofs).
- retry: work = ovl_start_creating_upper(ofs, workbasedir, &QSTR(name)).
- if work.d_inode exists:
  - if persist: return work (e.g. "index").
  - if !retried: ovl_workdir_cleanup(ofs, workbasedir, mnt, work, 0); dput; retry.
  - else: -EEXIST.
- else: ovl_do_mkdir(ofs, dir, work, S_IFDIR|0).
- if d_really_is_negative(work): -EINVAL.
- ovl_do_remove_acl(work, POSIX_ACL_DEFAULT) — accept -ENODATA / -EOPNOTSUPP.
- ovl_do_remove_acl(work, POSIX_ACL_ACCESS) — same.
- ovl_do_notify_change(work, attr_mode_0).
- return work.

REQ-11: ovl_make_workdir(sb, ofs, upper_path):
- ovl_workdir_create(ofs, OVL_WORKDIR_NAME, /*persist=*/false).
- ofs.workdir = work.
- feature probes against work:
  - ovl_setxattr(ofs, work, OVL_XATTR_OPAQUE, "0", 1) ⟹ if -EOPNOTSUPP: ofs.noxattr = true; pr_warn fallbacks for redirect_dir → nofollow, metacopy → off.
  - ovl_removexattr(ofs, work, OVL_XATTR_OPAQUE).
  - ovl_check_rename_whiteout(ofs) — probe RENAME_WHITEOUT.
  - probe tmpfile support — ofs.tmpfile = true on success.
- if config.nfs_export ∧ !config.index: pr_warn "NFS export requires \"index=on\", falling back to nfs_export=off."

REQ-12: ovl_get_workdir(sb, ofs, upper_path, work_path):
- if !ovl_workdir_ok(work_path.dentry, upper_path.dentry): -EINVAL (must share fs, must not be subdir of upper).
- ofs.workbasedir = dget(work_path.dentry).
- ovl_setup_trap(sb, workbasedir, &ofs.workbasedir_trap, "workdir").
- ovl_inuse_trylock(workbasedir) — workdir_locked = true.
- ovl_report_in_use on conflict.
- ovl_make_workdir(sb, ofs, upper_path).
- if !ovl_should_sync(ofs) ∧ ofs.workdir: ovl_create_volatile_dirty(ofs).

REQ-13: ovl_get_indexdir(sb, ofs, oe, upper_path):
- index_dir = ovl_workdir_create(ofs, OVL_INDEXDIR_NAME, /*persist=*/true).
- ovl_setup_trap(sb, index_dir, &ofs.indexdir_trap, "indexdir").
- if index empty ∧ has-upper-origin: write OVL_XATTR_ORIGIN to index root for verifiable file-handle decoding.
- on incompatible-upper (uuid mismatch / origin xattr mismatch): -EINVAL pr_warn "incompatible upper".

REQ-14: ovl_get_upper(sb, ofs, layer0, upper_path):
- if upper_path.dentry.d_sb.s_op == &ovl_super_operations: pr_err "upperdir is in-use by an overlay"; -EBUSY.
- clone-private mnt from upper_path.mnt; MNT_INTERNAL.
- layer0.mnt = upper_mnt; layer0.fs = upper_sb_annotation; layer0.idx = 0; layer0.fsid = 0.
- ovl_inuse_trylock(upper_path.dentry); ofs.upperdir_locked = true; on conflict: ovl_report_in_use.
- ovl_setup_trap(sb, upper_path.dentry, &ofs.upperdir_trap, "upperdir").

REQ-15: ovl_get_layers(sb, ofs, ctx, layers):
- For each i in 1..ctx.nr:
  - l = &ctx.lower[i-1].
  - ovl_lower_dir(name, &l.path, ofs, &stack_depth) — probes file-handle support; on lack: fallback ofs.config.nfs_export=off, ofs.nofh=true (per ovl_check_layer).
  - layers[i].mnt = clone_private_mount(&l.path); MNT_INTERNAL; MNT_READONLY.
  - layers[i].fs = ofs.fs[fsid].
  - layers[i].idx = i.
  - layers[i].fsid = ovl_get_fsid(ofs, &l.path) (unique per underlying sb; 0 reserved for upper; bad_uuid layers share fsid 0 if !has_fsid).
  - layers[i].has_xwhiteouts as discovered.
  - ovl_setup_trap.
- update ofs.numlayer, ofs.numfs, ofs.numdatalayer.
- sb.s_stack_depth = max(layers[i].mnt.mnt_sb.s_stack_depth) + 1; if > FILESYSTEM_MAX_STACK_DEPTH: -EINVAL.

REQ-16: ovl_get_fsid(ofs, path):
- uuid = path.mnt.mnt_sb.s_uuid.
- if uuid is_null:
  - if (config.index ∨ config.nfs_export) ∧ already-used-as-lower:
    - pr_warn "uuid in non-single lower fs, falling back to xino=off,index=off,nfs_export=off."
    - config.xino = OFF; config.index = false; config.nfs_export = false.
- search ofs.fs[1..numfs] for matching uuid:
  - if found: return existing fsid.
  - else: alloc new ovl_sb, assign pseudo_dev, fsid++, return.

REQ-17: ovl_lower_uuid_ok(ofs, uuid):
- if !config.nfs_export ∧ !ovl_upper_mnt(ofs): return true (lower-only mount with no index/export — uuid irrelevant).
- accept null uuid for single-lower (special-cased).
- otherwise require unique non-null uuid per layer.

REQ-18: ovl_check_layer(sb, ofs, path, name, stack_depth):
- if path.dentry.d_sb == sb: -ELOOP.
- if path.dentry.d_sb.s_op == &ovl_super_operations: -EINVAL "filesystem on '%s' not supported as upperdir/lowerdir (overlay-on-overlay)".
- if (config.nfs_export ∨ config.index) ∧ !path.dentry.d_sb.s_export_op.encode_fh:
  - pr_warn "fs on '%s' does not support file handles, falling back to index=off,nfs_export=off."
  - config.index = false; config.nfs_export = false.
- track *stack_depth = max(*stack_depth, path.dentry.d_sb.s_stack_depth).

REQ-19: ovl_check_overlapping_layers(sb, ofs):
- For each pair (layer_i, layer_j) including upperdir/workdir/indexdir:
  - if vfs path of one is ancestor/descendant of another: -ELOOP.

REQ-20: ovl_set_d_op(sb):
- if config.upperdir ∨ any-lower-needs-revalidate:
  - sb.s_d_op = &ovl_dentry_operations (with .d_revalidate / .d_weak_revalidate hooks).
- else: sb.s_d_op = &ovl_dentry_operations_noop.

REQ-21: ovl_dentry_revalidate / _common / _weak / ovl_revalidate_real:
- For each ovl_path in lowerstack ∪ {upperdentry}: real = ovl_dentry_real_at(layer); d_revalidate(real, flags) (or weak); if any layer rejects ⟹ dentry invalid.

REQ-22: copy-up lifecycle (`ovl_copy_up_data`, in fs/overlayfs/copy_up.c):
- struct ovl_copy_up_ctx { ofs, parent, dentry, stat, lowerpath, destdir, destname, indexed, metacopy, … }.
- On first write to a lower file (or lower dir descendant before any dir-op):
  - ovl_copy_up_start (acquires oi.lock).
  - clone lower attrs → temp in workdir (or O_TMPFILE on supporting fs).
  - ovl_copy_up_data: if !metacopy ∨ data-modifying: open lower (read), open temp (write), iterate via vfs_copy_file_range / splice fallback; preserve size, mtime, atime.
  - ovl_copy_up_metadata: set xattrs (whitelisted), uid/gid/mode/POSIX-ACL, security.*, NLINK, ORIGIN xattr (file-handle of lower).
  - rename temp → upperdir under destname (workdir cleanup on failure).
  - if config.metacopy ∧ data-not-modified: write OVL_XATTR_METACOPY (sized struct ovl_metacopy with optional fsverity digest).
  - mark inode OVL_UPPER (and OVL_UPPERDATA if data copied).
  - ovl_copy_up_end (releases oi.lock).
- Subsequent writes go to upperdentry directly.

REQ-23: Per-trusted.overlay.* xattr semantics:
- `OVL_XATTR_OPAQUE` = "trusted.overlay.opaque" (or "user.overlay.opaque"): on a directory, mask the same name in lower layers (whiteout for whole subtree).
- `OVL_XATTR_REDIRECT` = "trusted.overlay.redirect": absolute path (under overlay root) where this dir's lower content lives — used when a directory is renamed across the upper-lower split.
- `OVL_XATTR_ORIGIN` = "trusted.overlay.origin": serialized file-handle of the lower-origin (for indexed lookup + verity + nfs_export).
- `OVL_XATTR_IMPURE` = "trusted.overlay.impure": directory contains copied-up entries — readdir must merge upper + lower (rather than rely on opaque short-circuit).
- `OVL_XATTR_NLINK` = "trusted.overlay.nlink": cached nlink (lower + index references).
- `OVL_XATTR_UPPER` = "trusted.overlay.upper": index-dir entry pointer for non-dir inodes.
- `OVL_XATTR_UUID` = "trusted.overlay.uuid": persistent uuid for export op.
- `OVL_XATTR_METACOPY` = "trusted.overlay.metacopy": metacopy marker (with struct ovl_metacopy payload optionally).
- `OVL_XATTR_PROTATTR` = "trusted.overlay.protattr": protected immutable/append flags.
- `OVL_XATTR_XWHITEOUT` = "trusted.overlay.xwhiteout": xattr-based whiteout (no chardev 0/0 mknod required).
- Xattr-handler dispatch in `ovl_xattr_handlers()` selects trusted.overlay.* (default) vs user.overlay.* (`userxattr` mount option, for unprivileged mounts in user namespace).

REQ-24: nfs_export indexed mode:
- Requires index=on (enforced; otherwise pr_warn fallback nfs_export=off).
- Requires every layer to support s_export_op->encode_fh (per REQ-18); otherwise fallback.
- ovl_export_operations.encode_fh constructs file_handle that records (lower-fid, upper-fid-if-present).
- decode_fh re-resolves via indexdir lookup (hashed by lower-fid).
- Readdir in indexed mode: walks indexdir entries (hashed origin xattrs), reconciles with upper readdir for OVL_IMPURE dirs.
- nfs_export incompatible with metacopy (per REQ-8 fallback).

REQ-25: ovl_put_super(sb):
- if ofs: ovl_free_fs(ofs).
  - ovl_free_fs releases: cap_lower-undone creator_cred, dput(workdir/workbasedir/indexdir), kfree(layers[0..].mnt private clones via mntput), kfree(lowerdirs[]), kfree(layers), kfree(ofs.fs[]), kfree(ofs).

REQ-26: ovl_sync_fs(sb, wait):
- ovl_sync_status(ofs) — returns -EIO if errseq fired on volatile mount.
- if !wait: return 0 (overlay defers to upper sync via SB_I_SKIP_SYNC walking).
- down_read(upper_sb.s_umount); sync_filesystem(upper_sb); up_read.

REQ-27: ovl_statfs(dentry, buf):
- ovl_path_real(root, &path) — picks upper-mnt if present else top lower.
- vfs_statfs(&path, buf).
- buf.f_namelen = ofs.namelen (min across layers).
- buf.f_type = OVERLAYFS_SUPER_MAGIC.
- if ovl_has_fsid(ofs): buf.f_fsid = uuid_to_fsid(sb.s_uuid).

REQ-28: ovl_init_uuid_xattr(sb, ofs, upper_path):
- if config.uuid in {ON, AUTO}: ensure upperdir has OVL_XATTR_UUID set; generate uuid_random if absent; populate sb.s_uuid for nfs_export.

REQ-29: ovl_xattr_handlers(ofs):
- Returns &ovl_trusted_xattr_handlers if !config.userxattr; else &ovl_user_xattr_handlers — both wrap a common ovl_other_xattr_get/set that delegates to upperdentry (after copy-up) or lower.

REQ-30: cap_lower CAP_SYS_RESOURCE:
- After fill_super, creator_cred.cap_effective drops CAP_SYS_RESOURCE so per-overlay-op never overrides disk quota / reserved-space on upper.

## Acceptance Criteria

- [ ] AC-1: mount with lowerdir=A:B and no upperdir: r/o mount, sb.s_root visible, both A and B merged.
- [ ] AC-2: mount with upperdir=U workdir=W lowerdir=L: r/w mount; workdir on same fs as upperdir validated.
- [ ] AC-3: workdir on different fs than upperdir ⟹ ovl_workdir_ok rejects; mount r/o or fails.
- [ ] AC-4: workdir initial xattr probe -EOPNOTSUPP ⟹ ofs.noxattr=true; redirect_dir falls back to nofollow; metacopy=off.
- [ ] AC-5: lowerdir on overlayfs (stacked overlay-on-overlay) ⟹ -EINVAL.
- [ ] AC-6: overlapping layer (lowerdir is subdir of upperdir or workdir) ⟹ -ELOOP.
- [ ] AC-7: index=on, nfs_export=on but a lower fs lacks encode_fh ⟹ index=off, nfs_export=off, pr_warn emitted.
- [ ] AC-8: metacopy=on with nfs_export=on ⟹ nfs_export=off, pr_warn.
- [ ] AC-9: write to a lower file triggers ovl_copy_up_data; upper now has full data + xattrs + OVL_XATTR_ORIGIN.
- [ ] AC-10: copy-up under metacopy=on without data-modifying write ⟹ upper has metadata + OVL_XATTR_METACOPY only.
- [ ] AC-11: directory rename across upper/lower: target directory carries OVL_XATTR_REDIRECT with original-lower-path.
- [ ] AC-12: directory with OVL_XATTR_OPAQUE: readdir does NOT include lower entries.
- [ ] AC-13: directory with OVL_XATTR_IMPURE: readdir merges upper + lower (with dedup against whiteouts).
- [ ] AC-14: userxattr=on: xattr names use user.overlay.* prefix; trusted.* writes by overlay-internal still rejected when not CAP_SYS_ADMIN.
- [ ] AC-15: volatile mount: upper sb_wb_err non-zero at mount ⟹ -EIO; subsequent err sets errseq for ovl_sync_status to return -EIO.
- [ ] AC-16: statfs returns f_namelen = ofs.namelen and f_type = OVERLAYFS_SUPER_MAGIC.
- [ ] AC-17: umount via ovl_put_super: all layer mnt's mntput, all dentries dput, ofs freed; underlying fs's not affected.
- [ ] AC-18: nfs_export readdir of indexed mode resolves a file_handle stably across umount/remount.
- [ ] AC-19: upperdir already in-use by another overlay mount ⟹ ovl_inuse_trylock fails; ovl_report_in_use; -EBUSY.
- [ ] AC-20: sb.s_stack_depth + 1 > FILESYSTEM_MAX_STACK_DEPTH ⟹ -EINVAL.

## Architecture

```
struct OvlConfig {
  upperdir: Option<String>,
  workdir: Option<String>,
  lowerdirs: Vec<String>,
  default_permissions: bool,
  redirect_mode: OvlRedirectMode,         // OFF / FOLLOW / NOFOLLOW / ON
  verity_mode: OvlVerityMode,
  index: bool,
  uuid: OvlUuidMode,                      // OFF / NULL / AUTO / ON
  nfs_export: bool,
  xino: OvlXinoMode,
  metacopy: bool,
  userxattr: bool,
  fsync_mode: OvlFsyncMode,               // includes VOLATILE
}

struct OvlSb {
  sb: *SuperBlock,
  pseudo_dev: DevT,
  bad_uuid: bool,
  is_lower: bool,
}

struct OvlLayer {
  mnt: *Vfsmount,                         // first member — relied on by free path
  trap: *Inode,
  fs: *OvlSb,
  idx: i32,
  fsid: i32,
  has_xwhiteouts: bool,
}

struct OvlPath { layer: &'static OvlLayer, dentry: *Dentry }

struct OvlEntry {
  __numlower: u32,
  __lowerstack: [OvlPath; __numlower],    // FAM via __counted_by
}

struct OvlFs {
  numlayer: u32, numfs: u32, numdatalayer: u32,
  layers: *[OvlLayer],                    // length numlayer+1
  fs: *[OvlSb],
  workbasedir: *Dentry, workdir: *Dentry,
  namelen: i64,
  config: OvlConfig,
  creator_cred: *Cred,
  tmpfile: bool, noxattr: bool, nofh: bool,
  upperdir_locked: bool, workdir_locked: bool,
  workbasedir_trap: *Inode, workdir_trap: *Inode,
  xino_mode: i32,
  last_ino: AtomicI64,
  whiteout: *Dentry, no_shared_whiteout: bool, whiteout_lock: Mutex,
  errseq: ErrseqT,
  casefold: bool,
}

struct OvlInode {
  // union (S_ISDIR vs S_ISREG)
  cache: *OvlDirCache,
  lowerdata_redirect: *String,
  redirect: *String,
  version: u64,
  flags: AtomicU64,                       // OVL_UPPERDATA | OVL_IMPURE | OVL_REDIRECT | OVL_METACOPY | OVL_WHITEOUTS | OVL_XWHITEOUT | ...
  vfs_inode: Inode,
  __upperdentry: *Dentry,
  oe: *OvlEntry,
  lock: Mutex,                            // serializes copy-up
}

enum OvlXattr { OPAQUE, REDIRECT, ORIGIN, IMPURE, NLINK, UPPER, UUID, METACOPY, PROTATTR, XWHITEOUT }
```

`OvlFs::fill_super(sb, fc) -> i32`:
1. if WARN_ON(fc.user_ns != current_user_ns()): return -EIO.
2. OvlFs::set_d_op(sb).
3. if !ofs.creator_cred: ofs.creator_cred = prepare_creds() ?? -ENOMEM.
4. with_ovl_creds(sb) { err = OvlFs::fill_super_creds(fc, sb) }.
5. on err: OvlFs::free_fs(ofs); sb.s_fs_info = None.
6. return err.

`OvlFs::fill_super_creds(fc, sb)`:
1. params_verify(ctx, &ofs.config)?
2. if ctx.nr == 0: pr_err "missing 'lowerdir'"; return -EINVAL.
3. layers = kzalloc::<OvlLayer>(ctx.nr + 1) ?? -ENOMEM.
4. ofs.config.lowerdirs = kcalloc(ctx.nr+1, sizeof(char*)).
5. ofs.config.lowerdirs[0] = take ctx.lowerdir_all (raw).
6. ofs.layers = layers; ofs.numlayer = 1.
7. sb.s_stack_depth = 0; sb.s_maxbytes = MAX_LFS_FILESIZE.
8. ofs.last_ino = 1.
9. if config.xino != OFF: ofs.xino_mode = BITS_PER_LONG - 32 (or fallback to OFF on 32-bit).
10. sb.s_op = &ovl_super_operations.
11. if config.upperdir.is_some():
    - require config.workdir; else pr_err + return -EINVAL.
    - OvlFs::get_upper(sb, ofs, &layers[0], &ctx.upper)?
    - if !ovl_should_sync: snapshot errseq; if errseq_check non-zero: return -EIO "Cannot mount volatile when upperdir has an unseen error".
    - OvlFs::get_workdir(sb, ofs, &ctx.upper, &ctx.work)?
    - if !ofs.workdir: sb.s_flags |= SB_RDONLY.
    - sb.s_stack_depth = upper.s_stack_depth.
    - sb.s_time_gran = upper.s_time_gran.
12. oe = OvlFs::get_lowerstack(sb, ctx, ofs, layers)?
13. if !ovl_upper_mnt(ofs): sb.s_flags |= SB_RDONLY.
14. if ovl_has_fsid(ofs) ∧ ovl_upper_mnt(ofs): OvlFs::init_uuid_xattr(sb, ofs, &ctx.upper).
15. if !ovl_force_readonly(ofs) ∧ config.index:
    - OvlFs::get_indexdir(sb, ofs, oe, &ctx.upper)?
    - if !ofs.workdir: sb.s_flags |= SB_RDONLY.
16. OvlFs::check_overlapping_layers(sb, ofs)?
17. if !ofs.workdir: config.index = false; if upper ∧ nfs_export: pr_warn + nfs_export=false.
18. if config.metacopy ∧ config.nfs_export: pr_warn + nfs_export=false.
19. sb.s_export_op = if config.nfs_export: &ovl_export_operations; else if !ofs.nofh: &ovl_export_fid_operations.
20. cap_lower(creator_cred.cap_effective, CAP_SYS_RESOURCE).
21. sb.s_magic = OVERLAYFS_SUPER_MAGIC.
22. sb.s_xattr = OvlFs::xattr_handlers(ofs).
23. sb.s_fs_info = ofs.
24. sb.s_flags |= SB_POSIXACL.
25. sb.s_iflags |= SB_I_SKIP_SYNC | SB_I_NOUMASK | SB_I_EVM_HMAC_UNSUPPORTED.
26. sb.s_root = OvlFs::get_root(sb, ctx.upper.dentry, oe) ?? -ENOMEM ⟹ out_free_oe.
27. return Ok.

`OvlFs::get_upper(sb, ofs, layer0, upper_path)`:
1. if upper_path.dentry.d_sb.s_op == &ovl_super_operations: pr_err; return -EBUSY (no overlay-on-overlay).
2. layer0.mnt = clone_private_mount(upper_path); MNT_INTERNAL.
3. layer0.fs = lookup-or-create OvlSb for upper sb.
4. layer0.idx = 0; layer0.fsid = 0.
5. ovl_inuse_trylock(upper_path.dentry); ofs.upperdir_locked = true; on conflict: report_in_use.
6. OvlFs::setup_trap(sb, upper_path.dentry, &ofs.upperdir_trap, "upperdir").

`OvlFs::get_workdir(sb, ofs, upper_path, work_path)`:
1. if !OvlFs::workdir_ok(work_path.dentry, upper_path.dentry): return -EINVAL.
2. ofs.workbasedir = dget(work_path.dentry).
3. OvlFs::setup_trap(workbasedir, &ofs.workbasedir_trap, "workdir").
4. ovl_inuse_trylock(workbasedir); ofs.workdir_locked = true.
5. OvlFs::make_workdir(sb, ofs, upper_path).

`OvlFs::make_workdir(sb, ofs, upper_path)`:
1. work = OvlFs::workdir_create(ofs, "work", /*persist=*/false)?
2. ofs.workdir = work.
3. probe xattr write OVL_XATTR_OPAQUE on workdir:
   - on -EOPNOTSUPP: ofs.noxattr = true; warn redirect_dir → nofollow; warn metacopy → off; config.metacopy = false.
4. remove probe xattr.
5. OvlFs::check_rename_whiteout(ofs).
6. probe O_TMPFILE: ofs.tmpfile = result.
7. if config.nfs_export ∧ !config.index: warn + nfs_export = false.

`OvlFs::get_layers(sb, ofs, ctx, layers)`:
1. for i in 1..=ctx.nr:
   - l = &ctx.lower[i-1].
   - OvlFs::lower_dir(name, &l.path, ofs, &mut stack_depth)?
   - layers[i].mnt = clone_private_mount(&l.path); MNT_INTERNAL | MNT_READONLY.
   - layers[i].fs = OvlFs::get_fsid(ofs, &l.path)?
   - layers[i].idx = i; layers[i].fsid = layers[i].fs.fsid_id.
   - OvlFs::setup_trap(layers[i].mnt.mnt_root, &layers[i].trap, "lowerdir").
2. ofs.numlayer += ctx.nr.
3. sb.s_stack_depth = stack_depth + 1; if > FILESYSTEM_MAX_STACK_DEPTH: -EINVAL.

`OvlFs::check_overlapping_layers(sb, ofs)`:
1. for (a, b) in all-pairs(layers ∪ {workdir, indexdir}):
   - if is_subdir(a.mnt.mnt_root, b.mnt.mnt_root) ∨ inverse: return -ELOOP.

`OvlFs::sync_fs(sb, wait) -> i32`:
1. ret = ovl_sync_status(ofs).
2. if ret < 0: return -EIO.
3. if !ret: return 0.
4. if !wait: return 0.
5. upper_sb = ovl_upper_mnt(ofs).mnt_sb.
6. down_read(&upper_sb.s_umount); ret = sync_filesystem(upper_sb); up_read.
7. return ret.

`OvlFs::statfs(dentry, buf)`:
1. ovl_path_real(sb.s_root, &path).
2. err = vfs_statfs(&path, buf).
3. on success: buf.f_namelen = ofs.namelen; buf.f_type = OVERLAYFS_SUPER_MAGIC.
4. if ovl_has_fsid(ofs): buf.f_fsid = uuid_to_fsid(sb.s_uuid).

`OvlInode::destroy(inode)`:
1. oi = OVL_I(inode).
2. dput(oi.__upperdentry).
3. ovl_stack_put(ovl_lowerstack(oi.oe), ovl_numlower(oi.oe)).
4. if S_ISDIR: ovl_dir_cache_free(inode).
5. else: kfree(oi.lowerdata_redirect).

`OvlInode::free(inode)`:
1. kfree(oi.redirect).
2. kfree(oi.oe).
3. mutex_destroy(&oi.lock).
4. kmem_cache_free(ovl_inode_cachep, oi).

`OvlCopyUp::data(ctx, temp)` (in copy_up.c):
1. err = ovl_copy_up_file(ofs, c.dentry, new_file, c.stat.size, /* metacopy */, ...).
2. on success: mark OVL_UPPERDATA; clear OVL_METACOPY if set.

`OvlFs::xattr_handlers(ofs) -> &'static [XattrHandler]`:
1. if config.userxattr: return &ovl_user_xattr_handlers.
2. else: return &ovl_trusted_xattr_handlers.

`xattr_to_name(ofs, ox) -> &'static str` (`ovl_xattr_table[ox][userxattr ? 1 : 0]`):
- OPAQUE   → "trusted.overlay.opaque"    /  "user.overlay.opaque"
- REDIRECT → "trusted.overlay.redirect"  /  "user.overlay.redirect"
- ORIGIN   → "trusted.overlay.origin"    /  "user.overlay.origin"
- IMPURE   → "trusted.overlay.impure"    /  "user.overlay.impure"
- NLINK    → "trusted.overlay.nlink"     /  "user.overlay.nlink"
- UPPER    → "trusted.overlay.upper"     /  "user.overlay.upper"
- UUID     → "trusted.overlay.uuid"      /  "user.overlay.uuid"
- METACOPY → "trusted.overlay.metacopy"  /  "user.overlay.metacopy"
- PROTATTR → "trusted.overlay.protattr"  /  "user.overlay.protattr"
- XWHITEOUT→ "trusted.overlay.xwhiteout" /  "user.overlay.xwhiteout"

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `numlayer_ge_1_after_upper` | INVARIANT | per-fill_super_creds: after get_upper success, ofs.numlayer ≥ 1 (layer[0] is upper). |
| `layer0_is_upper_or_null` | INVARIANT | per-ofs.layers: idx==0 ⟹ upper-mnt or null (lower-only mount). |
| `workdir_on_same_fs_as_upper` | INVARIANT | per-workdir_ok: ofs.workdir.d_sb == ovl_upper_mnt(ofs).mnt_sb. |
| `no_overlay_on_overlay` | INVARIANT | per-get_upper/lower_dir: upper.s_op != ovl_super_operations ∧ lower.s_op != ovl_super_operations. |
| `no_overlapping_layers` | INVARIANT | per-check_overlapping_layers: ∀ pair: !is_subdir(a, b). |
| `stack_depth_capped` | INVARIANT | per-get_layers: sb.s_stack_depth ≤ FILESYSTEM_MAX_STACK_DEPTH. |
| `nfs_export_requires_index` | INVARIANT | per-make_workdir / fill_super_creds: config.nfs_export ⟹ config.index ∨ !workdir-needed. |
| `nfs_export_requires_encode_fh` | INVARIANT | per-check_layer: nfs_export=true ⟹ every layer.s_export_op.encode_fh != NULL. |
| `metacopy_excludes_nfs_export` | INVARIANT | per-fill_super_creds: config.metacopy ∧ config.nfs_export ⟹ nfs_export disabled. |
| `userxattr_implies_unprivileged_namespace_safe` | INVARIANT | per-xattr_handlers: config.userxattr ⟹ user.overlay.* prefix selected. |
| `creator_cred_cap_sys_resource_dropped` | INVARIANT | per-fill_super_creds: creator_cred.cap_effective lacks CAP_SYS_RESOURCE post-mount. |
| `lowerdir_mnt_internal_and_readonly` | INVARIANT | per-get_layers: layers[i>0].mnt has MNT_INTERNAL ∧ MNT_READONLY. |
| `inuse_locks_paired_with_unlock_on_free` | INVARIANT | per-put_super: upperdir_locked ⟹ ovl_inuse_unlock invoked on free; same for workdir_locked. |
| `xino_off_on_32bit` | INVARIANT | per-fill_super_creds: BITS_PER_LONG==32 ⟹ xino_mode = OFF (cannot embed fsid). |
| `volatile_mount_errseq_observed` | INVARIANT | per-fill_super_creds: !ovl_should_sync ⟹ errseq snapshot taken; ovl_sync_fs surfaces -EIO. |
| `copy_up_serialized_by_oi_lock` | INVARIANT | per-do_copy_up: oi.lock held throughout temp creation, data copy, metadata, and rename. |

### Layer 2: TLA+

`fs/overlayfs/super.tla`:
- Per-mount + per-config-fallbacks + per-copy-up + per-redirect-rename + per-nfs-export-decode.
- Properties:
  - `safety_no_overlay_on_overlay` — per-mount: a layer with s_op == ovl_super_operations is rejected at mount.
  - `safety_workdir_same_fs` — per-mount: workdir.d_sb == upperdir.d_sb.
  - `safety_no_layer_loop` — per-check_overlapping_layers: graph of (mnt_root, ancestor) is acyclic across upper/work/index/lower.
  - `safety_lower_readonly` — per-get_layers: lower.mnt has MNT_READONLY (no writes propagate to lower).
  - `safety_copy_up_idempotent` — per-multi-writer: concurrent writers see oi.lock serialize; either both see upperdentry post-copy-up or one waits.
  - `safety_redirect_path_preserved` — per-rename-across-layer: target inode carries OVL_XATTR_REDIRECT = original lower path.
  - `safety_opaque_blocks_lower_readdir` — per-readdir: OVL_XATTR_OPAQUE on dir ⟹ lower entries absent.
  - `safety_impure_merges_readdir` — per-readdir: OVL_XATTR_IMPURE on dir ⟹ upper + lower entries union (whiteout-filtered).
  - `liveness_copy_up_terminates` — per-write-to-lower: oi.lock acquired ⟹ eventually upperdentry set ∨ error.
  - `liveness_mount_terminates` — per-fill_super_creds: returns 0 or specific error; never infinite-retry.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `OvlFs::fill_super_creds` post: sb.s_op == &ovl_super_operations ∧ sb.s_root != null ∨ err returned | `OvlFs::fill_super_creds` |
| `OvlFs::get_upper` post: layers[0].idx == 0 ∧ fsid == 0 ∧ MNT_INTERNAL set | `OvlFs::get_upper` |
| `OvlFs::get_workdir` post: ofs.workdir on same sb as ofs.layers[0].mnt | `OvlFs::get_workdir` |
| `OvlFs::make_workdir` post: noxattr ⟹ !redirect_follow ∧ !metacopy | `OvlFs::make_workdir` |
| `OvlFs::get_layers` post: ∀ i>0 layers[i].mnt MNT_READONLY | `OvlFs::get_layers` |
| `OvlFs::check_overlapping_layers` post: no pair (a,b) with is_subdir | `OvlFs::check_overlapping_layers` |
| `OvlInode::destroy` post: __upperdentry put, lowerstack put | `OvlInode::destroy` |
| `OvlFs::sync_fs(wait)` post: wait=1 ⟹ upper_sb sync_filesystem invoked under read s_umount | `OvlFs::sync_fs` |
| `OvlFs::statfs` post: buf.f_type == OVERLAYFS_SUPER_MAGIC ∧ buf.f_namelen == ofs.namelen | `OvlFs::statfs` |
| `OvlCopyUp::do_copy_up` post: success ⟹ OVL_UPPER set; metacopy ⟹ OVL_METACOPY set ∧ !OVL_UPPERDATA | `OvlCopyUp::do_copy_up` |

### Layer 4: Verus/Creusot functional

`Per-mount (upperdir + workdir + lowerdir=L1:L2:…) → ovl_get_upper → ovl_get_workdir + feature probes → ovl_get_layers → ovl_get_indexdir → check_overlapping_layers → ovl_get_root → per-VFS-op: lookup walks lowerstack ∪ {upper} ⟹ first match (with OVL_XATTR_OPAQUE short-circuit, OVL_XATTR_REDIRECT path-jump) → per-write: ovl_copy_up_data ⟹ upper has full file (or OVL_XATTR_METACOPY for metadata-only) → per-readdir: merge with whiteouts and OVL_XATTR_IMPURE handling → per-umount: ovl_put_super` semantic equivalence: per-Documentation/filesystems/overlayfs.rst (Linux 7.1.0-rc2 baseline).

## Hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

Overlayfs reinforcement:

- **Per-no-overlay-on-overlay** — defense against per-stacking-loop (s_stack_depth explosion + recursive sb lookup).
- **Per-FILESYSTEM_MAX_STACK_DEPTH cap** — defense against per-deep-stack stack-overflow in VFS recursion.
- **Per-no-overlapping-layers check** — defense against per-self-aliasing (upperdir within lowerdir ⟹ infinite copy-up).
- **Per-ovl_inuse_trylock on upperdir/workdir** — defense against per-double-mount of same upper (data divergence).
- **Per-trap-inodes installed for each layer** — defense against per-mount-the-same-overlay-as-lower.
- **Per-workdir on same fs as upperdir** — defense against per-rename-cross-fs failure mid-copy-up (workdir → upper rename must be atomic).
- **Per-clone_private_mount(lower) MNT_READONLY | MNT_INTERNAL** — defense against per-userspace-mutation of lower behind overlay's back.
- **Per-CAP_SYS_RESOURCE dropped from creator_cred** — defense against per-overlay-op evading disk-quota / reserved-space on upper.
- **Per-userxattr (CONFIG-gated) for unprivileged-overlay** — defense against per-trusted.* xattr-write requiring CAP_SYS_ADMIN.
- **Per-OVL_XATTR_OPAQUE / IMPURE / REDIRECT canonical handling** — defense against per-readdir-leaking-lower-after-rename.
- **Per-OVL_XATTR_ORIGIN file-handle verification** — defense against per-nfs_export-stale-handle-after-relayer.
- **Per-metacopy + verity (OVL_XATTR_METACOPY payload includes fsverity digest)** — defense against per-tampered-upper-data.
- **Per-volatile-mount errseq sampling** — defense against per-silent-data-loss after fsync skipped.
- **Per-RENAME_WHITEOUT probe + ovl_check_rename_whiteout** — defense against per-stale-whiteout after rename.
- **Per-noxattr-fallback for upper-fs-without-xattr** — defense against per-mount-fail-on-tmpfs-upper.
- **Per-MNT_INTERNAL on cloned upper/lower mounts** — defense against per-userspace-umount of internal mount.
- **Per-config.nfs_export gated on every-layer-supports-encode_fh** — defense against per-half-working-export (silent data corruption on decode).
- **Per-fallback warnings (pr_warn) on incompatible options** — defense against per-silent-misconfiguration.
- **Per-with_ovl_creds wrapping fill_super_creds** — defense against per-caller-cred-leakage into underlying-fs lookups.
- **Per-OVERLAYFS_SUPER_MAGIC distinct + sb.s_export_op vtables distinct (decodable vs file-id-only)** — defense against per-export-mode confusion.
- **Per-OVL_XATTR_PROTATTR for immutable/append flags** — defense against per-flag-stripping-on-copy-up.
- **Per-OVL_XATTR_XWHITEOUT on layers that cannot mknod char 0/0** — defense against per-no-whiteout-on-fat upper.
- **Per-casefolding propagated only when both upper supports + lower compatible** — defense against per-cross-layer-case-confusion.

## Open Questions

- Whether to model the redirect-dir / metacopy fast paths (`fs/overlayfs/namei.c`, `fs/overlayfs/dir.c`) in this Tier-3 or split to a dedicated `overlayfs/dir.md`. Current document references their xattr semantics only.
- Whether `userxattr` mode warrants its own state-machine in the TLA+ model versus a parameter.
- nfs_export encode/decode is referenced via `ovl_export_operations` but a separate `overlayfs/export.md` Tier-3 may be warranted given the file-handle marshalling complexity.

## Out of Scope

- fs/overlayfs/namei.c lookup walk + redirect resolution (covered separately if expanded)
- fs/overlayfs/dir.c readdir merge + whiteout + IMPURE (covered separately if expanded)
- fs/overlayfs/copy_up.c full data path (referenced; covered separately as `overlayfs/copy-up.md`)
- fs/overlayfs/file.c open/read/write per-VFS-op delegation (covered separately)
- fs/overlayfs/export.c nfs_export encode/decode (covered separately)
- fs/overlayfs/params.c mount-option parsing (covered separately)
- fs/overlayfs/inode.c per-inode setattr/getattr/permission (covered separately)
- fs/overlayfs/xattr.c xattr-handler get/set/list (referenced; covered separately)
- Implementation code
