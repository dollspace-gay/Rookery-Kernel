# Tier-3: kernel/bpf/inode.c — bpffs (BPF filesystem)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/bpf/00-overview.md
upstream-paths:
  - kernel/bpf/inode.c (~1097 lines)
  - kernel/bpf/preload/bpf_preload.h
  - include/linux/bpf.h (struct bpf_mount_opts)
  - include/uapi/linux/magic.h (BPF_FS_MAGIC = 0xcafe4a11)
-->

## Summary

The **BPF filesystem** ("bpffs", `magic = BPF_FS_MAGIC = 0xcafe4a11`) is a tiny in-memory pseudo-FS that provides a namespace for **pinning** BPF objects — programs, maps, and links — so they outlive the process that created them and can be discovered by name. Conventionally mounted at `/sys/fs/bpf` (`fs_kobj` mount-point pre-created at boot). Each pinned object lives as a single inode whose `inode_operations` pointer encodes its kind (`bpf_prog_iops` / `bpf_map_iops` / `bpf_link_iops` / `bpf_dir_iops` for directories) and whose `i_private` holds the kernel object pointer. Two syscalls reach this layer: **BPF_OBJ_PIN** (`bpf_obj_pin_user`) creates a dentry under a bpffs directory pointing at a fd's underlying object; **BPF_OBJ_GET** (`bpf_obj_get_user`) walks a path and re-installs a new fd of the correct kind from the inode. Mount-time **BPF token delegation** (`delegate_cmds`, `delegate_maps`, `delegate_progs`, `delegate_attachs` mount opts, parsed via BTF enum names) builds the namespace within which `BPF_TOKEN_CREATE` may mint a delegated capability bundle. Critical for: unprivileged-with-token BPF workflows, container runtimes (CRI-O, containerd, systemd-nspawn), kubelet, kubernetes CNI dataplane, libbpf-skel pinning patterns, bpftool `pin` / `show pinned`.

This Tier-3 covers `kernel/bpf/inode.c` (~1097 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enum bpf_type` (UNSPEC/PROG/MAP/LINK) | per-inode kind tag | `BpfType` |
| `bpf_any_get(raw, type)` | per-refcount inc by type | `Bpffs::any_get` |
| `bpf_any_put(raw, type)` | per-refcount dec by type | `Bpffs::any_put` |
| `bpf_fd_probe_obj(ufd, *type)` | per-fd kind probe (map→prog→link) | `Bpffs::fd_probe_obj` |
| `bpf_inode_type(inode, *type)` | per-inode→type reverse map | `Bpffs::inode_type` |
| `bpf_get_inode(sb, dir, mode)` | per-inode alloc + ownership init | `Bpffs::get_inode` |
| `bpf_dentry_finalize(dentry, inode, dir)` | per-d_make_persistent + mtime | `Bpffs::dentry_finalize` |
| `bpf_mkdir(idmap, dir, dentry, mode)` | per-`mkdir` op | `Bpffs::mkdir` |
| `bpf_lookup(dir, dentry, flags)` | per-lookup (reject names with `.`) | `Bpffs::lookup` |
| `bpf_symlink(idmap, dir, dentry, target)` | per-symlink op | `Bpffs::symlink` |
| `bpf_mkobj_ops(dentry, mode, raw, iops, fops)` | per-mknod for prog/map/link | `Bpffs::mkobj_ops` |
| `bpf_mkprog` / `bpf_mkmap` / `bpf_mklink` | per-kind mknod | `Bpffs::mk{prog,map,link}` |
| `bpf_obj_do_pin(path_fd, pathname, raw, type)` | per-create + vfs_mkobj | `Bpffs::obj_do_pin` |
| `bpf_obj_pin_user(ufd, path_fd, pathname)` | per-BPF_OBJ_PIN entry | `Bpffs::obj_pin_user` |
| `bpf_obj_do_get(path_fd, pathname, *type, flags)` | per-path walk + inode→object | `Bpffs::obj_do_get` |
| `bpf_obj_get_user(path_fd, pathname, flags)` | per-BPF_OBJ_GET entry | `Bpffs::obj_get_user` |
| `__get_prog_inode(inode, type)` | per-inode→prog (with type check) | `Bpffs::get_prog_inode` |
| `bpf_prog_get_type_path(name, type)` | per-kernel-path prog get | `Bpffs::prog_get_type_path` |
| `bpf_iter_link_pin_kernel(parent, name, link)` | per-preload iter pin | `Bpffs::iter_link_pin_kernel` |
| `bpf_destroy_inode(inode)` | per-inode dtor (put underlying obj) | `Bpffs::destroy_inode` |
| `bpf_super_ops` | per-super_operations | `Bpffs::SUPER_OPS` |
| `bpf_show_options(m, root)` | per-`/proc/mounts` formatter | `Bpffs::show_options` |
| `bpf_parse_param(fc, param)` | per-fsparam parser | `Bpffs::parse_param` |
| `bpf_fill_super(sb, fc)` | per-super init | `Bpffs::fill_super` |
| `bpf_init_fs_context(fc)` / `bpf_get_tree(fc)` / `bpf_kill_super(sb)` / `bpf_free_fc(fc)` | per-context lifecycle | `Bpffs::{init_fs_context,get_tree,kill_super,free_fc}` |
| `bpf_context_ops` / `bpf_fs_type` | per-fs-registration tables | `Bpffs::{CONTEXT_OPS,FS_TYPE}` |
| `find_bpffs_btf_enums(info)` | per-vmlinux-BTF enum lookup (cmd/map/prog/attach) | `Bpffs::find_btf_enums` |
| `find_btf_enum_const(btf, enum_t, prefix, str, *value)` | per-enum string lookup | `Bpffs::find_enum_const` |
| `seq_print_delegate_opts(m, name, btf, enum_t, prefix, mask, any)` | per-delegate-mask renderer | `Bpffs::print_delegate_opts` |
| `populate_bpffs(parent)` | per-bpf_preload iter linkage | `Bpffs::populate` |
| `bpf_preload_ops` / `bpf_preload_mod_get/put()` | per-preload-module ref | `Bpffs::preload_*` |
| `bpf_init()` (`fs_initcall`) | per-mountpoint + register_filesystem | `Bpffs::init` |
| `bpffs_map_fops` / `bpffs_map_seq_ops` | per-`cat`-able map view | `Bpffs::MAP_FOPS` / `MAP_SEQ_OPS` |
| `bpffs_obj_fops` (open=-EIO) | per-default fops (no direct read) | `Bpffs::OBJ_FOPS` |

## Compatibility contract

REQ-1: enum bpf_type:
- `BPF_TYPE_UNSPEC = 0` (sentinel, set initially in lookups).
- `BPF_TYPE_PROG` — inode.i_op == `bpf_prog_iops`; `i_private = (struct bpf_prog *)`.
- `BPF_TYPE_MAP` — inode.i_op == `bpf_map_iops`; `i_private = (struct bpf_map *)`.
- `BPF_TYPE_LINK` — inode.i_op == `bpf_link_iops`; `i_private = (struct bpf_link *)`.
- Directories use `bpf_dir_iops` (distinguished separately).

REQ-2: Per-type refcount adapters:
- `bpf_any_get(raw, BPF_TYPE_PROG)` → `bpf_prog_inc(raw)`.
- `bpf_any_get(raw, BPF_TYPE_MAP)` → `bpf_map_inc_with_uref(raw)`.
- `bpf_any_get(raw, BPF_TYPE_LINK)` → `bpf_link_inc(raw)`.
- `bpf_any_put(raw, BPF_TYPE_PROG)` → `bpf_prog_put(raw)`.
- `bpf_any_put(raw, BPF_TYPE_MAP)` → `bpf_map_put_with_uref(raw)`.
- `bpf_any_put(raw, BPF_TYPE_LINK)` → `bpf_link_put(raw)`.
- Default ⟹ `WARN_ON_ONCE(1)`.

REQ-3: fd → object kind probe (`bpf_fd_probe_obj`):
- Try `bpf_map_get_with_uref(ufd)` ⟹ `*type = BPF_TYPE_MAP`.
- Else try `bpf_prog_get(ufd)` ⟹ `*type = BPF_TYPE_PROG`.
- Else try `bpf_link_get_from_fd(ufd)` ⟹ `*type = BPF_TYPE_LINK`.
- All-fail ⟹ -EINVAL via `ERR_PTR`.
- Ordering is observable: a fd that aliases multiple types (impossible by construction since each kind has distinct file_ops) is resolved as map first; library code should not depend on this beyond the documented kinds.

REQ-4: inode → object kind probe (`bpf_inode_type`):
- Compare `inode.i_op` pointer identity:
  - `&bpf_prog_iops` ⟹ PROG.
  - `&bpf_map_iops` ⟹ MAP.
  - `&bpf_link_iops` ⟹ LINK.
  - else ⟹ `-EACCES` with `*type = UNSPEC`.

REQ-5: bpf_get_inode (factory):
- mode bits S_IFMT must be one of S_IFDIR, S_IFREG, S_IFLNK; else -EINVAL.
- `new_inode(sb)` ⟹ -ENOSPC on alloc failure.
- `i_ino = get_next_ino()`; `simple_inode_init_ts(inode)`.
- `inode_init_owner(&nop_mnt_idmap, inode, dir, mode)` (no per-user-ns idmap).

REQ-6: bpf_mkdir / bpf_lookup / bpf_symlink:
- mkdir: `bpf_get_inode(dir.sb, dir, mode|S_IFDIR)`; `i_op = &bpf_dir_iops`; `i_fop = &simple_dir_operations`; `inc_nlink(inode)` + `inc_nlink(dir)`; finalize.
- lookup: rejects `.` in component name when `dir.i_mode & S_IALLUGO` is non-zero (reserves dot-prefixed names for `populate_bpffs` special files); falls back to `simple_lookup`.
- symlink: `kstrdup(target, GFP_USER|__GFP_NOWARN)`; alloc inode with mode `S_IRWXUGO|S_IFLNK`; `i_op = &simple_symlink_inode_operations`; `i_link = link`.

REQ-7: bpf_dir_iops:
- `lookup = bpf_lookup`.
- `mkdir = bpf_mkdir`.
- `symlink = bpf_symlink`.
- `rmdir = simple_rmdir`.
- `rename = simple_rename`.
- `link = simple_link`.
- `unlink = simple_unlink`.

REQ-8: bpf_mkobj_ops (mknod for BPF objects):
- `dir = dentry.d_parent.d_inode`.
- `inode = bpf_get_inode(dir.sb, dir, mode)`.
- `inode.i_op = iops` (one of bpf_{prog,map,link}_iops).
- `inode.i_fop = fops` (one of `bpffs_obj_fops`, `bpffs_map_fops`, `bpf_iter_fops`).
- `inode.i_private = raw` (object pointer with caller-held ref).
- `bpf_dentry_finalize(dentry, inode, dir)`.

REQ-9: Per-kind mkobj:
- `bpf_mkprog(dentry, mode, arg)` → `bpf_mkobj_ops(dentry, mode, arg, &bpf_prog_iops, &bpffs_obj_fops)`.
- `bpf_mkmap(dentry, mode, arg)` → `bpf_mkobj_ops(dentry, mode, arg, &bpf_map_iops, bpf_map_support_seq_show(map) ? &bpffs_map_fops : &bpffs_obj_fops)`.
- `bpf_mklink(dentry, mode, arg)` → `bpf_mkobj_ops(dentry, mode, arg, &bpf_link_iops, bpf_link_is_iter(link) ? &bpf_iter_fops : &bpffs_obj_fops)`.

REQ-10: bpf_obj_do_pin (per-path creation):
- `dentry = start_creating_user_path(path_fd, pathname, &path, 0)` ⟹ propagate err.
- `dir = d_inode(path.dentry)`; require `dir.i_op == &bpf_dir_iops` else -EPERM.
- `mode = S_IFREG | ((S_IRUSR|S_IWUSR) & ~current_umask())`.
- `security_path_mknod(&path, dentry, mode, 0)`.
- Dispatch by `type`: `vfs_mkobj(dentry, mode, bpf_mkprog|bpf_mkmap|bpf_mklink, raw)`.
- Always `end_creating_path(&path, dentry)`.

REQ-11: bpf_obj_pin_user (BPF_OBJ_PIN entry from syscall.c):
- `raw = bpf_fd_probe_obj(ufd, &type)`; propagate err.
- `ret = bpf_obj_do_pin(path_fd, pathname, raw, type)`.
- On `ret != 0`: `bpf_any_put(raw, type)` (probe took a ref; release on failure).

REQ-12: bpf_obj_do_get (per-path lookup):
- `user_path_at(path_fd, pathname, LOOKUP_FOLLOW, &path)` ⟹ propagate err.
- `inode = d_backing_inode(path.dentry)`.
- `path_permission(&path, ACC_MODE(flags))` ⟹ propagate err.
- `bpf_inode_type(inode, type)` ⟹ propagate err (-EACCES if not BPF inode).
- `raw = bpf_any_get(inode.i_private, *type)` — bumps refcount of the underlying object.
- On success: `touch_atime(&path)`.
- `path_put(&path)` (always).

REQ-13: bpf_obj_get_user (BPF_OBJ_GET entry from syscall.c):
- `f_flags = bpf_get_file_flag(flags)`.
- `raw = bpf_obj_do_get(path_fd, pathname, &type, f_flags)`.
- Branch on type:
  - PROG → `bpf_prog_new_fd(raw)`.
  - MAP → `bpf_map_new_fd(raw, f_flags)`.
  - LINK → require `f_flags == O_RDWR` else -EINVAL; `bpf_link_new_fd(raw)`.
  - else ⟹ -ENOENT.
- On fd-install failure: `bpf_any_put(raw, type)`.

REQ-14: __get_prog_inode / bpf_prog_get_type_path (kernel-internal):
- `inode_permission(&nop_mnt_idmap, inode, MAY_READ)` first.
- Reject if `i_op == &bpf_map_iops` or `&bpf_link_iops` ⟹ -EINVAL.
- Require `i_op == &bpf_prog_iops` else -EACCES.
- `security_bpf_prog(prog)` ⟹ propagate err.
- `bpf_prog_get_ok(prog, &type, false)` ⟹ -EINVAL on type mismatch.
- `bpf_prog_inc(prog)`.
- `bpf_prog_get_type_path`: `kern_path(name, LOOKUP_FOLLOW, &path)`; `__get_prog_inode`; `touch_atime` on success; `path_put`. `EXPORT_SYMBOL` so other in-kernel users (network filter classifier, etc.) can resolve by path string.

REQ-15: bpffs_map_fops (cat-able map view):
- `open = bpffs_map_open`: alloc `struct map_iter { void *key; bool done; }`; `seq_open(file, &bpffs_map_seq_ops)`; attach iter via `m.private`.
- `read = seq_read`.
- `release = bpffs_map_release`: free `map_iter` then `seq_release`.
- Seq ops:
  - `start(m, pos)`: returns SEQ_START_TOKEN if pos==0 else `iter.key`.
  - `next(m, v, pos)`: walks `map.ops.map_get_next_key(map, prev_key, key)` under rcu_read_lock; sets `iter.done = true` on exhaustion.
  - `show(m, v)`: prints header warning on first call; else `map.ops.map_seq_show_elem(map, key, m)`.
- Output is debug-only; userspace tools (bpftool, libbpf) must use BPF_MAP_LOOKUP_ELEM for real reads. Header banner: `"# WARNING!! The output is for debug purpose only"`, `"# WARNING!! The output format will change"`.

REQ-16: bpffs_obj_fops (default for prog / non-iter link / map without seq_show):
- `open = bpffs_obj_open` returns -EIO.
- No other ops — opening a pinned prog or link via `open(2)` is intentionally rejected; userspace must go through BPF_OBJ_GET.

REQ-17: bpf_iter_link_pin_kernel (preload):
- `dentry = simple_start_creating(parent, name)`.
- `bpf_mkobj_ops(dentry, S_IFREG|S_IRUSR, link, &bpf_link_iops, &bpf_iter_fops)`.
- `simple_done_creating(dentry)`.
- Used by `populate_bpffs` to install kernel-built iterators at mount time.

REQ-18: bpf_destroy_inode (per-inode dtor):
- If `S_ISLNK(inode.i_mode)`: `kfree(inode.i_link)`.
- If `bpf_inode_type(inode, &type)` succeeds: `bpf_any_put(inode.i_private, type)` — drops the pin-held ref to the underlying object.
- `free_inode_nonrcu(inode)`.

REQ-19: bpf_super_ops:
- `statfs = simple_statfs`.
- `drop_inode = inode_just_drop`.
- `show_options = bpf_show_options`.
- `destroy_inode = bpf_destroy_inode`.

REQ-20: Mount option parser (bpf_fs_parameters / bpf_parse_param):
- OPT_UID (fsparam_u32): `make_kuid(current_user_ns(), v)` ⟹ -EINVAL on invalid or unmappable.
- OPT_GID (fsparam_u32): `make_kgid(current_user_ns(), v)` ⟹ same.
- OPT_MODE (fsparam_u32oct): `opts.mode = v & S_IALLUGO`.
- OPT_DELEGATE_CMDS / _MAPS / _PROGS / _ATTACHS (fsparam_string):
  - Parse colon-separated list. Each token:
    - `"any"` ⟹ `msk |= ~0ULL`.
    - else lookup via `find_btf_enum_const(vmlinux_btf, enum_t, prefix, token, &val)` ⟹ `msk |= 1ULL << val`.
    - else `kstrtou64(token, 0, &msk)` (raw hex/dec fallback) ⟹ propagate err.
  - Setting any non-zero mask ⟹ `capable(CAP_SYS_ADMIN)` else -EPERM.
  - `*delegate_msk |= msk` (additive across param invocations).
- Unknown option ⟹ ignored (legacy compat: traditionally all bpf mount opts were ignored).
- Bad value ⟹ `invalfc(fc, "Bad value for '%s'", param->key)`.

REQ-21: bpf_show_options (/proc/mounts formatter):
- If `inode.i_uid != GLOBAL_ROOT_UID`: emit `,uid=%u`.
- If `inode.i_gid != GLOBAL_ROOT_GID`: emit `,gid=%u`.
- If `mode != S_IRWXUGO`: emit `,mode=%o`.
- For each non-zero delegate_* mask: `seq_print_delegate_opts(m, name, btf, enum_t, prefix, mask, any_mask)` where `any_mask = (1ULL << __MAX_BPF_*) - 1`.
- Renderer emits lower-case enum names with prefix stripped, separated by `:`; falls back to hex `0x...` for unknown bits when vmlinux BTF unavailable.

REQ-22: bpf_fill_super:
- `if fc.user_ns != &init_user_ns && !capable(CAP_SYS_ADMIN)` ⟹ -EPERM (no userns mount without admin).
- `simple_fill_super(sb, BPF_FS_MAGIC, bpf_rfiles)` where `bpf_rfiles[] = { {""} }` (empty template).
- `sb.s_op = &bpf_super_ops`.
- `inode = sb.s_root.d_inode`; `inode.i_uid = opts.uid`; `inode.i_gid = opts.gid`; `inode.i_op = &bpf_dir_iops`; clear S_IALLUGO bits.
- `populate_bpffs(sb.s_root)` (preload iters, ignored on err).
- `inode.i_mode |= S_ISVTX | opts.mode` (sticky bit always set).

REQ-23: bpf_init_fs_context / bpf_get_tree / bpf_kill_super / bpf_free_fc:
- `init_fs_context`: `opts = kzalloc(struct bpf_mount_opts)`; defaults `mode = S_IRWXUGO`, `uid = current_fsuid()`, `gid = current_fsgid()`, all delegate masks = 0; `fc.s_fs_info = opts`; `fc.ops = &bpf_context_ops`.
- `get_tree`: `get_tree_nodev(fc, bpf_fill_super)`.
- `kill_super`: `opts = sb.s_fs_info`; `kill_anon_super(sb)`; `kfree(opts)`.
- `free_fc`: `kfree(fc.s_fs_info)`.

REQ-24: bpf_fs_type registration:
- `.name = "bpf"`.
- `.init_fs_context = bpf_init_fs_context`.
- `.parameters = bpf_fs_parameters`.
- `.kill_sb = bpf_kill_super`.
- `.fs_flags = FS_USERNS_MOUNT` (mountable from non-init userns, gated by `bpf_fill_super`'s capable check).

REQ-25: bpf_init (`fs_initcall`):
- `sysfs_create_mount_point(fs_kobj, "bpf")` — creates `/sys/fs/bpf` directory entry.
- `register_filesystem(&bpf_fs_type)`.
- On failure: `sysfs_remove_mount_point(fs_kobj, "bpf")`.

REQ-26: populate_bpffs (preload):
- mutex_lock(`bpf_preload_lock`).
- `bpf_preload_mod_get()`: if `!bpf_preload_ops`, `request_module("bpf_preload")`; then `try_module_get(bpf_preload_ops.owner)`.
- `bpf_preload_ops.preload(objs[BPF_PRELOAD_LINKS])`.
- For each obj: `bpf_link_inc(objs[i].link)`; `bpf_iter_link_pin_kernel(parent, objs[i].link_name, objs[i].link)`; on err `bpf_link_put`.
- `bpf_preload_mod_put`.
- All errors swallowed (best-effort; mount succeeds even without preload).

REQ-27: BPF token mount-time delegation (cross-ref to syscall.c BPF_TOKEN_CREATE):
- bpffs mount's `delegate_cmds`, `delegate_maps`, `delegate_progs`, `delegate_attachs` masks form the **maximum** capability set a token derived from this mount may grant.
- `BPF_TOKEN_CREATE` opens the bpffs (via `attr.token_create.bpffs_fd`), intersects requested with these mount masks, and pins a `struct bpf_token` whose `allowed_cmds` ⊆ `opts.delegate_cmds`, etc.
- Mount-time `capable(CAP_SYS_ADMIN)` check ensures only admin (or admin-equivalent in mount namespace) can set delegate masks; this is the trust anchor for delegation.

## Acceptance Criteria

- [ ] AC-1: `mount -t bpf bpf /sys/fs/bpf` succeeds in init userns without options.
- [ ] AC-2: `mount -t bpf bpf /mnt` from non-init userns without CAP_SYS_ADMIN returns -EPERM.
- [ ] AC-3: `mount -t bpf -o delegate_cmds=any bpf /mnt` without CAP_SYS_ADMIN returns -EPERM.
- [ ] AC-4: `mount -t bpf -o delegate_cmds=prog_load:map_create bpf /mnt` sets `opts.delegate_cmds` bitmask correctly via BTF enum lookup.
- [ ] AC-5: `BPF_OBJ_PIN` of a prog fd at `/sys/fs/bpf/foo` creates an inode with `i_op == bpf_prog_iops` and `i_private == prog`.
- [ ] AC-6: `BPF_OBJ_GET` on `/sys/fs/bpf/foo` returns a new prog fd referring to the same `struct bpf_prog`.
- [ ] AC-7: `BPF_OBJ_PIN` with parent dir not under bpffs (e.g. `/tmp/foo`) returns -EPERM.
- [ ] AC-8: `BPF_OBJ_GET` with `flags=O_RDONLY` on a LINK inode returns -EINVAL.
- [ ] AC-9: `BPF_OBJ_GET` on a path whose inode is not a BPF inode returns -EACCES.
- [ ] AC-10: `mkdir /sys/fs/bpf/sub` succeeds and is mountable as parent for further pins.
- [ ] AC-11: `open("/sys/fs/bpf/pinned_prog", O_RDONLY)` returns -EIO (must use BPF_OBJ_GET).
- [ ] AC-12: `cat /sys/fs/bpf/pinned_hashmap` (when map supports seq_show) emits debug header + key/value lines.
- [ ] AC-13: `umount /sys/fs/bpf` decrements refcount on every pinned object (`bpf_destroy_inode` → `bpf_any_put`).
- [ ] AC-14: Filename containing `.` (e.g. `foo.bar`) is rejected by `bpf_lookup` with -EPERM under non-zero-mode dir.
- [ ] AC-15: `/proc/mounts` displays bpf mount with `delegate_cmds=prog_load:map_create` etc. when set.

## Architecture

```
const BPF_FS_MAGIC: u32 = 0xcafe4a11;

enum BpfType { Unspec = 0, Prog, Map, Link }

struct BpfMountOpts {
  uid: Kuid,
  gid: Kgid,
  mode: u16,                  // S_IALLUGO subset
  delegate_cmds: u64,         // bitmap over enum bpf_cmd
  delegate_maps: u64,         // bitmap over enum bpf_map_type
  delegate_progs: u64,        // bitmap over enum bpf_prog_type
  delegate_attachs: u64,      // bitmap over enum bpf_attach_type
}

// Per-inode dispatch is by inode_operations pointer identity:
const BPF_DIR_IOPS:  InodeOperations = { lookup, mkdir, symlink, rmdir, rename, link, unlink };
const BPF_PROG_IOPS: InodeOperations = { };  // empty — pointer identity is the type tag
const BPF_MAP_IOPS:  InodeOperations = { };  // empty
const BPF_LINK_IOPS: InodeOperations = { };  // empty

const BPF_SUPER_OPS: SuperOperations = {
  statfs: simple_statfs,
  drop_inode: inode_just_drop,
  show_options: bpf_show_options,
  destroy_inode: bpf_destroy_inode,
};

const BPF_FS_TYPE: FileSystemType = {
  name: "bpf",
  init_fs_context: bpf_init_fs_context,
  parameters: BPF_FS_PARAMETERS,
  kill_sb: bpf_kill_super,
  fs_flags: FS_USERNS_MOUNT,
  owner: THIS_MODULE,
};
```

`Bpffs::obj_pin_user(ufd, path_fd, pathname) -> Result<(), Errno>`:
1. /* probe fd to determine object kind (map → prog → link) */
2. let (raw, ty) = `Bpffs::fd_probe_obj(ufd)` — each probe takes its own ref.
3. /* create dentry + inode under bpffs */
4. ret = `Bpffs::obj_do_pin(path_fd, pathname, raw, ty)`.
5. /* on failure release the ref taken by probe */
6. if ret != 0: `Bpffs::any_put(raw, ty)`.
7. return ret.

`Bpffs::obj_do_pin(path_fd, pathname, raw, ty) -> Result<(), Errno>`:
1. dentry = `start_creating_user_path(path_fd, pathname, &path, 0)`.
2. dir = `d_inode(path.dentry)`.
3. /* must be inside bpffs */
4. if dir.i_op != &BPF_DIR_IOPS ⟹ -EPERM.
5. mode = S_IFREG | ((S_IRUSR | S_IWUSR) & !current_umask()).
6. `security_path_mknod(&path, dentry, mode, 0)`.
7. match ty:
   - Prog ⟹ `vfs_mkobj(dentry, mode, Bpffs::mkprog, raw)`.
   - Map ⟹ `vfs_mkobj(dentry, mode, Bpffs::mkmap, raw)`.
   - Link ⟹ `vfs_mkobj(dentry, mode, Bpffs::mklink, raw)`.
   - else ⟹ -EPERM.
8. `end_creating_path(&path, dentry)`.
9. return ret.

`Bpffs::obj_get_user(path_fd, pathname, flags) -> Result<i32, Errno>`:
1. f_flags = `bpf_get_file_flag(flags)`.
2. /* path walk + reverse lookup */
3. let (raw, ty) = `Bpffs::obj_do_get(path_fd, pathname, f_flags)`.
4. /* install new fd of correct kind */
5. fd = match ty:
   - Prog ⟹ `bpf_prog_new_fd(raw)`.
   - Map ⟹ `bpf_map_new_fd(raw, f_flags)`.
   - Link ⟹ if f_flags != O_RDWR ⟹ -EINVAL; else `bpf_link_new_fd(raw)`.
   - else ⟹ -ENOENT.
6. if fd < 0: `Bpffs::any_put(raw, ty)`.
7. return fd.

`Bpffs::obj_do_get(path_fd, pathname, flags) -> Result<(*const c_void, BpfType), Errno>`:
1. `user_path_at(path_fd, pathname, LOOKUP_FOLLOW, &path)`.
2. inode = `d_backing_inode(path.dentry)`.
3. `path_permission(&path, ACC_MODE(flags))`.
4. ty = `Bpffs::inode_type(inode)` — -EACCES if not BPF inode.
5. raw = `Bpffs::any_get(inode.i_private, ty)` — bumps refcount.
6. on ok: `touch_atime(&path)`.
7. `path_put(&path)`.
8. return Ok((raw, ty)).

`Bpffs::fill_super(sb, fc) -> Result<(), Errno>`:
1. /* userns gate */
2. if fc.user_ns != &init_user_ns && !capable(CAP_SYS_ADMIN) ⟹ -EPERM.
3. `simple_fill_super(sb, BPF_FS_MAGIC, &[{ "" }])`.
4. sb.s_op = &BPF_SUPER_OPS.
5. inode = sb.s_root.d_inode.
6. inode.i_uid = opts.uid; inode.i_gid = opts.gid.
7. inode.i_op = &BPF_DIR_IOPS.
8. inode.i_mode &= !S_IALLUGO.
9. `Bpffs::populate(sb.s_root)` — best-effort preload pin.
10. inode.i_mode |= S_ISVTX | opts.mode.

`Bpffs::parse_param(fc, param) -> Result<(), Errno>`:
1. result = `fs_parse(fc, BPF_FS_PARAMETERS, param, &result)`.
2. match opt:
   - OPT_UID/GID: validate kuid_has_mapping / kgid_has_mapping in fc.user_ns.
   - OPT_MODE: opts.mode = v & S_IALLUGO.
   - OPT_DELEGATE_*:
     - Look up vmlinux BTF for `bpf_cmd` / `bpf_map_type` / `bpf_prog_type` / `bpf_attach_type` enum.
     - Tokenize value on `:` separators.
     - For each token: `"any"` ⟹ `msk |= ~0ULL`; else BTF enum lookup (case-insensitive, prefix-stripped); else `kstrtou64` raw fallback.
     - If msk != 0 ⟹ require `capable(CAP_SYS_ADMIN)` else -EPERM.
     - *delegate_msk |= msk.
   - default ⟹ ignore (legacy compat).

`Bpffs::destroy_inode(inode)`:
1. if S_ISLNK(inode.i_mode): kfree(inode.i_link).
2. if let Ok(ty) = `Bpffs::inode_type(inode)`: `Bpffs::any_put(inode.i_private, ty)`.
3. `free_inode_nonrcu(inode)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pin_under_bpffs_only` | INVARIANT | per-obj_do_pin: parent dir.i_op == &BPF_DIR_IOPS else -EPERM. |
| `inode_type_pointer_identity` | INVARIANT | per-inode_type: discriminated by i_op pointer identity; no aliasing. |
| `fd_probe_obj_takes_one_ref` | INVARIANT | per-fd_probe_obj: success path takes exactly one ref of correct kind. |
| `pin_failure_releases_probe_ref` | INVARIANT | per-obj_pin_user: failure path calls any_put(raw, type). |
| `get_inode_to_obj_refcount_balanced` | INVARIANT | per-obj_get_user: any_get bumps ref; fd-install failure calls any_put. |
| `link_get_requires_o_rdwr` | INVARIANT | per-obj_get_user: LINK type requires f_flags == O_RDWR else -EINVAL. |
| `inode_destroy_puts_underlying_obj` | INVARIANT | per-bpf_destroy_inode: i_private's underlying object reference is released exactly once. |
| `delegate_msk_requires_cap_sys_admin` | INVARIANT | per-parse_param: non-zero delegate mask requires CAP_SYS_ADMIN. |
| `mount_in_non_init_userns_requires_admin` | INVARIANT | per-fill_super: fc.user_ns != init_user_ns ⟹ CAP_SYS_ADMIN. |
| `lookup_rejects_dot_in_name` | INVARIANT | per-bpf_lookup: dir mode & S_IALLUGO ∧ '.' in name ⟹ -EPERM. |
| `bpffs_obj_open_returns_eio` | INVARIANT | per-bpffs_obj_fops: open(2) ⟹ -EIO (force BPF_OBJ_GET path). |
| `symlink_link_kfreed_on_destroy` | INVARIANT | per-bpf_destroy_inode: S_ISLNK ⟹ kfree(i_link). |

### Layer 2: TLA+

`kernel/bpf/inode.tla`:
- Per-mount → per-mkdir → per-pin → per-get → per-unlink → per-umount lifecycle.
- Properties:
  - `safety_pin_takes_one_ref` — per-pin: object refcount increases by exactly 1.
  - `safety_unpin_drops_one_ref` — per-unlink (vfs_unlink → bpf_destroy_inode): refcount decreases by exactly 1.
  - `safety_get_independent_of_pin` — per-get: returned fd has its own ref; closing fd does not affect pin.
  - `safety_token_mask_subset_of_mount` — per-BPF_TOKEN_CREATE on bpffs: token.allowed_* ⊆ opts.delegate_*.
  - `safety_no_cross_kind_install` — per-get: returned fd's underlying kind == inode's recorded kind.
  - `liveness_each_op_terminates` — per-op: completes in finite VFS steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Bpffs::fd_probe_obj` post: Ok ⟹ exactly one of {map_get_with_uref, prog_get, link_get_from_fd} succeeded | `Bpffs::fd_probe_obj` |
| `Bpffs::obj_do_pin` post: Ok ⟹ new dentry under bpffs parent with i_private = raw and correct i_op | `Bpffs::obj_do_pin` |
| `Bpffs::obj_pin_user` post: Err ⟹ raw's refcount unchanged from caller's view (probe ref released) | `Bpffs::obj_pin_user` |
| `Bpffs::obj_do_get` post: Ok ⟹ raw points to inode.i_private with bumped refcount; atime touched | `Bpffs::obj_do_get` |
| `Bpffs::obj_get_user` post: Ok ⟹ fd holds 1 ref; Err ⟹ no leaked ref | `Bpffs::obj_get_user` |
| `Bpffs::parse_param(DELEGATE_*)` post: Ok ⟹ opts.delegate_msk only adds bits permitted by vmlinux BTF enum range | `Bpffs::parse_param` |
| `Bpffs::fill_super` post: sb.s_op == &BPF_SUPER_OPS; root inode.i_op == &BPF_DIR_IOPS; S_ISVTX set | `Bpffs::fill_super` |
| `Bpffs::destroy_inode` post: i_private released via correct kind-specific put | `Bpffs::destroy_inode` |
| `Bpffs::inode_type` post: Ok ⟹ ty corresponds to i_op identity in {prog, map, link}_iops | `Bpffs::inode_type` |
| `Bpffs::show_options` post: emits each delegate_* token iff bit set; falls back to hex if BTF missing | `Bpffs::show_options` |

### Layer 4: Verus/Creusot functional

`Per-BPF_OBJ_PIN(ufd, pathname) → fd_probe_obj → vfs_mkobj(dentry, bpf_mk{prog,map,link}) → inode.i_private = raw with kind-specific i_op` and `Per-BPF_OBJ_GET(pathname) → user_path_at → inode_type via i_op pointer → any_get → kind-specific new_fd` semantic equivalence: per-`Documentation/bpf/bpffs.rst`, per-`include/uapi/linux/bpf.h` BPF_OBJ_PIN/BPF_OBJ_GET, per-bpftool `pin` / `show pinned` selftests (`tools/testing/selftests/bpf/`).

## Hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

bpffs reinforcement:

- **Per-`dir.i_op == &bpf_dir_iops` precondition for pin** — defense against per-cross-fs pin into a real filesystem.
- **Per-`security_path_mknod` LSM hook** — defense against per-policy bypass during pin (SELinux file_t, AppArmor path globs, BPF LSM).
- **Per-`inode_type` pointer-identity discriminator** — defense against per-type-confusion (cannot forge a "prog" inode from a "map" via mode bits or extended attrs).
- **Per-`bpffs_obj_open = -EIO`** — defense against per-direct-read of pinned object bytes (forces all access through BPF_OBJ_GET → verified fd ops).
- **Per-fd_probe_obj reference taken before publishing dentry** — defense against per-pin-then-disappear (object cannot be freed between probe and mkobj).
- **Per-pin failure releases probe ref via `bpf_any_put`** — defense against per-pin-fail leak.
- **Per-`bpf_destroy_inode` releases underlying object exactly once** — defense against per-umount UAF and per-double-put.
- **Per-`capable(CAP_SYS_ADMIN)` for non-init-userns mount** — defense against per-unprivileged-user-mount creating delegate-capable bpffs.
- **Per-`capable(CAP_SYS_ADMIN)` for any non-zero delegate_* mount option** — defense against per-unprivileged-token-mint.
- **Per-BTF enum lookup for delegate_* names** — defense against per-typo (cannot accidentally grant a wrong cmd by misspelling; falls through to numeric only if explicit `kstrtou64` succeeds).
- **Per-`bpf_lookup` rejects names containing `.`** — defense against per-namespace-poisoning of dot-prefixed reserved names used by `populate_bpffs`.
- **Per-LINK BPF_OBJ_GET requires O_RDWR** — defense against per-readonly-link-misuse (links inherently need update/detach permissions).
- **Per-`S_ISVTX` (sticky bit) always set on bpffs root** — defense against per-cross-uid unlink of another user's pinned object.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `BPF_OBJ_PIN` / `BPF_OBJ_GET` thread a user-supplied `pathname` through `getname` and back into `kern_path_create`; whitelist the strncpy_from_user / d_path window so an attacker cannot use a crafted bpffs path argument to leak adjacent dentry / mount fields across the user boundary.
- **PAX_KERNEXEC** — bpffs inode ops vector (`bpf_dir_iops`, `bpf_prog_iops`, `bpf_map_iops`, `bpf_link_iops`) lives in `.rodata`; enforce W^X so a heap-overflow into an adjacent superblock cannot rewrite `.lookup` to a controlled function pointer.
- **PAX_RANDKSTACK** — the pin/unpin path runs under VFS locks with arbitrary user-controlled component depth; randomized kstack offset shrinks the ROP window across the `vfs_mkobj` → `bpf_iop_mkdir`/`bpf_mkobj` callchain.
- **PAX_REFCOUNT** — `inode->i_private` holds raw `bpf_map`/`bpf_prog`/`bpf_link` refs counted by refcount_t; saturating overflow trap prevents `BPF_OBJ_GET` storm racing against `BPF_OBJ_PIN` unlink from underflowing to a fresh-but-freed object.
- **PAX_MEMORY_SANITIZE** — bpffs inode evict (`bpf_evict_inode`) must zero `inode->i_private` after the drop-ref so a subsequent slab reuse of the same `struct inode` cannot expose the freed bpf object pointer to a lookup racing the unlink.
- **PAX_UDEREF** — `BPF_OBJ_PIN`/`_OBJ_GET` argument vectors live in userspace `bpf_attr`; SMAP/PAN must be active across the whole `bpf_obj_pin_uattr` → `bpf_obj_do_pin` chain, not just at syscall entry.
- **PAX_RAP/kCFI** — `inode_operations->lookup` / `->mkdir` / `->unlink` indirect calls from VFS must verify the kCFI tag matches `bpf_dir_iops`; defense against confused-deputy where a corrupted mount could swap the iops vector for one accepting non-bpffs payloads.
- **GRKERNSEC_HIDESYM** — `seq_show_fdinfo` for bpffs pinned objects must hide the underlying `struct bpf_map *` / `struct bpf_prog *` kernel pointers from non-CAP_SYSLOG readers; expose id only.
- **GRKERNSEC_DMESG** — bpffs WARN_ON paths (orphaned inode, ref imbalance on evict) must not leak raw kernel pointers into dmesg readable by non-CAP_SYSLOG.
- **bpffs CAP_BPF mount gate** — `bpf_fill_super` should refuse mount unless the caller holds CAP_BPF in the userns owning the mount, even when `fsopen` is permitted; defense against per-namespace bpffs being used as a side-channel pinning surface.
- **BPF_OBJ_PIN PAX_USERCOPY path-walk window** — the `pathname` copy from user must be sized against PATH_MAX with strict overflow rejection so an oversized pin path cannot drive a slab overflow in `getname_flags`.
- **Reserved-name dot-prefix lockdown** — `bpf_lookup` already rejects `.`-containing names; harden by also refusing components matching `populate_bpffs` reserved prefixes (`.preserve_static_offset`, etc.) to prevent namespace poisoning even from CAP_BPF.
- **Rationale** — bpffs is the persistence layer for BPF objects across process lifetimes; without grsec gating an attacker who briefly obtains CAP_BPF can pin a prog/map under a long-lived bpffs path and keep its kernel state alive after cap drop. The combined W^X + kCFI + refcount-saturation regime forces an attacker to break multiple independent mitigations before they can convert a bpffs race into kernel control.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `bpf_sys_bpf` syscall surface (BPF_OBJ_PIN / _OBJ_GET dispatch entry points) — covered in `syscall.md` Tier-3.
- BPF token semantics (BPF_TOKEN_CREATE, `bpf_token_capable`, `bpf_token_allow_*`) — covered in `syscall.md` Tier-3 + Tier-3 token doc if expanded.
- BTF parsing for delegate enum lookup (`btf_find_by_name_kind`, `btf_type_by_id`, `btf_enum`) — covered in `btf.md` Tier-3.
- `bpf_iter_fops` (BPF iterator file ops) — covered in iterator Tier-3 if expanded.
- `bpf_preload.ko` userspace-mode-driver — covered in `preload/` Tier-3 if expanded.
- VFS primitives (`simple_lookup`, `simple_rename`, `simple_unlink`, `vfs_mkobj`, `start_creating_user_path`, `end_creating_path`, `user_path_at`, `path_permission`) — covered in fs/ Tier-3.
- Per-map / per-prog / per-link refcount internals (`bpf_prog_inc/put`, `bpf_map_inc_with_uref/put_with_uref`, `bpf_link_inc/put`) — covered in their respective Tier-3 docs.
- Implementation code.
