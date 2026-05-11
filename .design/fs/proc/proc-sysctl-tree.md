# Tier-3: fs/proc/proc_sysctl.c — /proc/sys tree (ctl_table / ctl_dir / ctl_table_root)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/proc/00-overview.md
upstream-paths:
  - fs/proc/proc_sysctl.c (~1726 lines)
  - include/linux/sysctl.h (struct ctl_table, ctl_table_header, ctl_table_root, ctl_dir, ctl_node, ctl_table_set)
  - kernel/sysctl.c (per-subsys default proc_handler set: proc_dointvec, proc_dostring, ...)
  - Documentation/admin-guide/sysctl/*
-->

## Summary

`fs/proc/proc_sysctl.c` is the procfs-side of the `/proc/sys/*` tree. It owns the per-`ctl_table_header` lifecycle, the per-`ctl_dir` rb-tree of named children, the per-set / per-root namespacing that lets each network namespace see its own `/proc/sys/net/*`, the dentry-ops that invalidate caches when a sysctl is unregistered, and the per-`proc_handler` dispatch that turns a write into a typed update of kernel state. Per-`register_sysctl` / `register_sysctl_sz` / `register_sysctl_mount_point` / `__register_sysctl_table` / `__register_sysctl_init`: build a path under `/proc/sys/<path>` and attach a `ctl_table` array. Per-`unregister_sysctl_table` / `drop_sysctl_table`: drain readers via `start_unregistering`'s completion handshake, invalidate dcache, then refcount-drop. Per-`proc_sys_lookup`: rb-tree find_entry under `sysctl_lock`, follow `S_IFLNK` net-namespace links, mint procfs inodes. Per-`proc_sys_readdir`: walk the rb-tree under per-`use_table` / `unuse_table` ref accounting. Per-`proc_sys_call_handler`: kbuf-alloc + BPF cgroup hook + per-`table->proc_handler` invocation. Per-named-children link tree (`new_links` / `insert_links` / `put_links` / `xlate_dir`): every namespaced sysctl set gets a shadow symlink chain in the root set so `/proc/sys/net/...` is reachable from the root mount-namespace even when each netns owns its private table.

This Tier-3 covers `fs/proc/proc_sysctl.c` (~1726 lines). The legacy global `kernel/sysctl.c` table and the per-handler implementations (`proc_dointvec`, `proc_dostring`, ...) are referenced as the call destination but live in their own Tier-3 (`proc-sysctl.md`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ctl_table` | per-entry: procname/data/maxlen/mode/proc_handler/extra1/extra2/type/poll | `procfs::sysctl::CtlTable` |
| `struct ctl_table_header` | per-table refcount+rb-node carrier | `procfs::sysctl::CtlTableHeader` |
| `struct ctl_node` | per-entry rb_node + back-pointer to header | `procfs::sysctl::CtlNode` |
| `struct ctl_dir` | per-directory: header + `root` (rb_root of children) | `procfs::sysctl::CtlDir` |
| `struct ctl_table_set` | per-namespace table-set: `dir`, `is_seen` | `procfs::sysctl::CtlTableSet` |
| `struct ctl_table_root` | per-namespace root: `default_set`, `lookup`, `permissions`, `set_ownership` | `procfs::sysctl::CtlTableRoot` |
| `sysctl_table_root` | per-global root | `procfs::sysctl::SYSCTL_TABLE_ROOT` |
| `root_table[]` | per-global synthetic dir-only `ctl_table` | `procfs::sysctl::ROOT_TABLE` |
| `sysctl_mount_point[]` | per-`register_sysctl_mount_point` empty marker | `procfs::sysctl::SYSCTL_MOUNT_POINT` |
| `sysctl_lock` (spinlock) | per-tree mutator + reader-count gate | `procfs::sysctl::SYSCTL_LOCK` |
| `sysctl_is_perm_empty_ctl_header` / `set_perm_empty` / `clear_perm_empty` | per-`SYSCTL_TABLE_TYPE_PERMANENTLY_EMPTY` flag-trio | `procfs::sysctl::perm_empty_*` |
| `proc_sys_poll_notify(poll)` | per-write wake of `ctl_table_poll` waiters | `procfs::sysctl::poll_notify` |
| `namecmp(n1,l1,n2,l2)` | per-rb-tree key compare (memcmp + len-tiebreak) | `procfs::sysctl::namecmp` |
| `find_entry(phead, dir, name, namelen)` | per-rb-tree find under sysctl_lock | `procfs::sysctl::find_entry` |
| `insert_entry(head, entry)` / `erase_entry` | per-rb-tree mutate | `procfs::sysctl::insert_entry` / `erase_entry` |
| `init_header(head, root, set, node, table, size)` | per-header bring-up | `procfs::sysctl::init_header` |
| `erase_header(head)` | per-header rb-tree purge | `procfs::sysctl::erase_header` |
| `insert_header(dir, header)` | per-attach: parent up-ref + insert_links + insert_entry-loop | `procfs::sysctl::insert_header` |
| `use_table(p)` / `unuse_table(p)` | per-reader ref account | `procfs::sysctl::use_table` / `unuse_table` |
| `start_unregistering(p)` | per-drain via completion + dcache invalidation | `procfs::sysctl::start_unregistering` |
| `sysctl_head_grab(head)` / `sysctl_head_finish` | per-readside head-ref bracket | `procfs::sysctl::head_grab` / `head_finish` |
| `lookup_header_set(root)` | per-root `lookup` hook → per-namespace set | `procfs::sysctl::lookup_header_set` |
| `lookup_entry(phead, dir, name, namelen)` | per-`proc_sys_lookup` rb-find with use_table | `procfs::sysctl::lookup_entry` |
| `first_usable_entry(node)` / `first_entry(dir,...)` / `next_entry(...)` | per-readdir rb iteration | `procfs::sysctl::iter_*` |
| `test_perm(mode, op)` / `sysctl_perm(head, table, op)` | per-mode UNIX-bit check (root-not-blanket-allowed) | `procfs::sysctl::perm` |
| `proc_sys_make_inode(sb, head, table)` | per-VFS inode mint + `sibling_inodes` linkage | `procfs::sysctl::make_inode` |
| `proc_sys_evict_inode(inode, head)` | per-evict: unlink + count-- + kfree_rcu | `procfs::sysctl::evict_inode` |
| `grab_header(inode)` | per-readside inode → head + grab | `procfs::sysctl::grab_header` |
| `proc_sys_lookup(dir, dentry, flags)` | per-VFS lookup entrypoint | `procfs::sysctl::vfs_lookup` |
| `proc_sys_call_handler(iocb, iter, write)` | per-read/write dispatch | `procfs::sysctl::call_handler` |
| `proc_sys_read` / `proc_sys_write` | per-iter wrapper | `procfs::sysctl::read_iter` / `write_iter` |
| `proc_sys_open` | per-open snapshot of `table->poll->event` | `procfs::sysctl::open` |
| `proc_sys_poll` | per-`ctl_table_poll` poll | `procfs::sysctl::poll` |
| `proc_sys_fill_cache` / `proc_sys_link_fill_cache` / `scan` / `proc_sys_readdir` | per-readdir | `procfs::sysctl::readdir_*` |
| `proc_sys_permission` / `proc_sys_setattr` / `proc_sys_getattr` | per-VFS attr | `procfs::sysctl::permission` / `setattr` / `getattr` |
| `proc_sys_file_operations` / `proc_sys_dir_file_operations` / `proc_sys_inode_operations` / `proc_sys_dir_operations` | per-VFS ops tables | `procfs::sysctl::FOPS_*` / `IOPS_*` |
| `proc_sys_revalidate` / `proc_sys_delete` / `proc_sys_compare` | per-dentry ops | `procfs::sysctl::dentry_*` |
| `proc_sys_dentry_operations` | per-dentry ops table | `procfs::sysctl::DENTRY_OPS` |
| `find_subdir(dir, name, namelen)` | per-subdir rb find | `procfs::sysctl::find_subdir` |
| `new_dir(set, name, namelen)` | per-subdir allocate (header+node+table+name) | `procfs::sysctl::new_dir` |
| `get_subdir(dir, name, namelen)` | per-subdir find-or-create with retry on lock-drop | `procfs::sysctl::get_subdir` |
| `xlate_dir(set, dir)` | per-set dir-translation for link tree | `procfs::sysctl::xlate_dir` |
| `sysctl_follow_link(phead, pentry)` | per-`S_IFLNK` namespace traversal | `procfs::sysctl::follow_link` |
| `sysctl_err(path, table, fmt, ...)` | per-validation error printer | `procfs::sysctl::err` |
| `sysctl_check_table_array` / `sysctl_check_table` | per-register validation | `procfs::sysctl::check_table_*` |
| `new_links(dir, head)` / `get_links(dir, header, link_root)` / `insert_links(head)` / `put_links(header)` | per-named-children link tree | `procfs::sysctl::links_*` |
| `sysctl_mkdir_p(dir, path)` | per-`mkdir -p` path walk | `procfs::sysctl::mkdir_p` |
| `__register_sysctl_table(set, path, table, size)` | per-leaf register entrypoint | `procfs::sysctl::register_table_inner` |
| `register_sysctl_sz(path, table, size)` | per-default-set register | `procfs::sysctl::register_sz` |
| `register_sysctl_mount_point(path)` | per-empty-dir register | `procfs::sysctl::register_mount_point` |
| `__register_sysctl_init(path, table, name, size)` | per-`__init` non-failing register | `procfs::sysctl::register_init` |
| `unregister_sysctl_table(header)` | per-tree teardown | `procfs::sysctl::unregister_table` |
| `drop_sysctl_table(header)` | per-header nreg-- with parent cascade | `procfs::sysctl::drop_table` |
| `setup_sysctl_set(set, root, is_seen)` / `retire_sysctl_set` | per-namespace bring-up / teardown | `procfs::sysctl::setup_set` / `retire_set` |
| `proc_sys_init()` | per-`proc_sys_root` `proc_mkdir("sys", NULL)` + `sysctl_init_bases` | `procfs::sysctl::init` |
| `sysctl_aliases[]` / `sysctl_find_alias` / `sysctl_is_alias` | per-cmdline-alias table (hardlockup_all_cpu_backtrace, hung_task_panic, numa_zonelist_order, softlockup_all_cpu_backtrace) | `procfs::sysctl::ALIASES` |
| `process_sysctl_arg(param, val, _, arg)` / `do_sysctl_args()` | per-cmdline `sysctl.foo=bar` writer | `procfs::sysctl::cmdline_args` |
| `sysctl_is_seen(p)` | per-`set->is_seen` namespace gate | `procfs::sysctl::is_seen` |
| `proc_sys_invalidate_dcache(head)` | per-unregister dcache purge | `procfs::sysctl::invalidate_dcache` |

## Compatibility contract

REQ-1: struct ctl_table:
- `procname`: filename under /proc/sys (NULL terminates legacy arrays; modern register uses explicit size).
- `data`: typed pointer the handler reads/writes.
- `maxlen`: byte capacity at `data`.
- `mode`: UNIX permission bits + S_IFDIR / S_IFREG / S_IFLNK; must be ≤ S_IRUGO|S_IWUGO for files.
- `type`: SYSCTL_TABLE_TYPE_DEFAULT or SYSCTL_TABLE_TYPE_PERMANENTLY_EMPTY.
- `proc_handler`: per-write/read function (proc_dointvec, proc_dostring, proc_dointvec_minmax, proc_doulongvec_minmax, proc_douintvec, proc_douintvec_minmax, proc_dou8vec_minmax, proc_dobool, proc_dointvec_jiffies, proc_dointvec_userhz_jiffies, proc_dointvec_ms_jiffies, proc_doulongvec_ms_jiffies_minmax, custom).
- `extra1`, `extra2`: per-handler bounds.
- `poll`: optional `ctl_table_poll` for waiters.

REQ-2: struct ctl_table_header:
- `ctl_table`, `ctl_table_size`, `ctl_table_arg`.
- `used` (active readers), `count` (total refs including inodes), `nreg` (registrations).
- `unregistering` (NULL | completion-pointer | ERR_PTR(-EINVAL) sentinel).
- `root`, `set`, `parent`.
- `node` (per-entry ctl_node array tail-allocated).
- `inodes` hlist (dcache children to invalidate on unregister).
- `rcu` head for `kfree_rcu`.

REQ-3: struct ctl_dir:
- Embedded `header` (its `ctl_table[0].procname` is the dir name; mode = S_IFDIR|S_IRUGO|S_IXUGO).
- `root`: rb_root of children's ctl_nodes.

REQ-4: struct ctl_table_set:
- `dir`: per-set root ctl_dir.
- `is_seen(set) -> int`: per-set visibility predicate (netns: tasks not in this netns return 0).

REQ-5: struct ctl_table_root:
- `default_set`: per-root default set (global root has the global default_set).
- `lookup(root) -> *set`: per-root namespace-aware set picker.
- `permissions(head, table) -> int`: optional override of `table->mode`.
- `set_ownership(head, *uid, *gid)`: optional uid/gid override (default GLOBAL_ROOT_UID / _GID).

REQ-6: sysctl_table_root + root_table:
- `root_table[0]` is a synthetic dir-only entry with `procname = ""`, `mode = S_IFDIR|S_IRUGO|S_IXUGO`.
- `sysctl_table_root.default_set.dir.header`: count=1, nreg=1, ctl_table=root_table, ctl_table_arg=root_table, root=&sysctl_table_root, set=&sysctl_table_root.default_set.

REQ-7: sysctl_lock (DEFINE_SPINLOCK):
- Guards: `dir.root` rb-trees, `head->used`, `head->count`, `head->nreg`, `head->unregistering`, `head->inodes`, `head->parent`.
- All mutators acquire it; many readers (`use_table` etc.) require it held.

REQ-8: namecmp(n1, l1, n2, l2):
- `cmp = memcmp(n1, n2, min(l1, l2))`.
- If 0: `cmp = l1 - l2`.

REQ-9: find_entry(phead, dir, name, namelen):
- lockdep_assert_held(sysctl_lock).
- Walk dir->root rb-tree: at each node, `entry = &head->ctl_table[ctl_node - head->node]`; compare via namecmp.
- On match: `*phead = head`; return entry.
- Else NULL.

REQ-10: insert_entry(head, entry):
- Compute `node = &head->node[entry - head->ctl_table].node`.
- Walk parent dir's rb-tree to find slot; on duplicate procname: pr_err "sysctl duplicate entry: <dir>/<procname>"; return -EEXIST.
- rb_link_node + rb_insert_color into `head->parent->root`.

REQ-11: init_header(head, root, set, node, table, table_size):
- Set ctl_table, ctl_table_size, ctl_table_arg, used=0, count=1, nreg=1, unregistering=NULL, root, set, parent=NULL, node.
- INIT_HLIST_HEAD(&head->inodes).
- If node: for each entry in table: `node->header = head; node++`.
- If table == sysctl_mount_point: set perm-empty type.

REQ-12: insert_header(dir, header):
- If dir is perm-empty: return -EROFS.
- If header is perm-empty:
  - If dir->root not empty: return -EINVAL.
  - Mark dir perm-empty.
- dir_h->nreg++.
- header->parent = dir.
- err = insert_links(header).
- If err: jump fail_links.
- For each entry: insert_entry(header, entry); on err: erase_header + put_links + clear perm-empty if applicable + drop_sysctl_table(dir).

REQ-13: use_table / unuse_table:
- use_table: lockdep_assert sysctl_lock; if p->unregistering: return 0; else `p->used++; return 1`.
- unuse_table: if `--p->used == 0` ∧ p->unregistering: `complete(p->unregistering)`.

REQ-14: start_unregistering(p):
- lockdep_assert sysctl_lock.
- If `p->used`:
  - init_completion(&wait); p->unregistering = &wait; spin_unlock; wait_for_completion(&wait).
- Else: `p->unregistering = ERR_PTR(-EINVAL)` (any non-NULL sentinel); spin_unlock.
- proc_sys_invalidate_dcache(p) — `proc_invalidate_siblings_dcache(&head->inodes, &sysctl_lock)`.
- spin_lock; erase_header(p).

REQ-15: sysctl_head_grab / sysctl_head_finish:
- grab: BUG_ON(!head); spin_lock; if !use_table(head): head = ERR_PTR(-ENOENT); spin_unlock; return.
- finish: if !head return; spin_lock; unuse_table; spin_unlock.

REQ-16: lookup_header_set(root):
- If root->lookup: return root->lookup(root).
- Else: return &root->default_set.

REQ-17: lookup_entry(phead, dir, name, namelen):
- spin_lock; entry = find_entry(&head, dir, name, namelen); if entry ∧ use_table(head): *phead = head; else entry = NULL.
- spin_unlock; return entry.

REQ-18: first_usable_entry(node) / first_entry / next_entry:
- first_usable_entry: walk rb_next-chain from node, return first ctl_node whose header is `use_table`-able.
- first_entry: spin_lock; first_usable_entry(rb_first(dir->root)); spin_unlock; derive entry = &head->ctl_table[ctl_node - head->node].
- next_entry: spin_lock; unuse_table(*phead); ctl_node = first_usable_entry(rb_next(&ctl_node->node)); spin_unlock; derive next or {NULL, NULL}.

REQ-19: test_perm(mode, op):
- If euid == GLOBAL_ROOT_UID: mode >>= 6.
- Else if in_egroup_p(GLOBAL_ROOT_GID): mode >>= 3.
- If `op & ~mode & (MAY_READ|MAY_WRITE|MAY_EXEC) == 0`: return 0.
- Else: return -EACCES.

REQ-20: sysctl_perm(head, table, op):
- If root->permissions: mode = root->permissions(head, table); else mode = table->mode.
- return test_perm(mode, op).

REQ-21: proc_sys_make_inode(sb, head, table):
- inode = new_inode; i_ino = get_next_ino.
- spin_lock; if head->unregistering: iput + return ERR_PTR(-ENOENT).
- ei->sysctl = head; ei->sysctl_entry = table.
- hlist_add_head_rcu(&ei->sibling_inodes, &head->inodes).
- head->count++.
- spin_unlock.
- simple_inode_init_ts; i_mode = table->mode.
- Per-S_ISDIR: i_mode |= S_IFDIR; i_op = dir_operations; i_fop = dir_file_operations; if perm-empty: make_empty_dir_inode.
- Else: i_mode |= S_IFREG; i_op = inode_operations; i_fop = file_operations.
- i_uid = GLOBAL_ROOT_UID; i_gid = GLOBAL_ROOT_GID; root->set_ownership? override.

REQ-22: proc_sys_evict_inode(inode, head):
- spin_lock; hlist_del_init_rcu(&PROC_I(inode)->sibling_inodes); if !--head->count: kfree_rcu(head, rcu); spin_unlock.

REQ-23: proc_sys_lookup(dir, dentry, flags):
- head = grab_header(dir); if IS_ERR: return.
- ctl_dir = container_of(head, struct ctl_dir, header).
- p = lookup_entry(&h, ctl_dir, name->name, name->len); if !p: out.
- If S_ISLNK(p->mode): ret = sysctl_follow_link(&h, &p); on err: out.
- inode = proc_sys_make_inode(sb, h ?: head, p).
- err = d_splice_alias_ops(inode, dentry, &proc_sys_dentry_operations).
- out: head_finish(h?); head_finish(head); return err.

REQ-24: proc_sys_call_handler(iocb, iter, write):
- inode = file_inode(iocb->ki_filp); head = grab_header(inode); table = PROC_I(inode)->sysctl_entry; count = iov_iter_count(iter).
- If IS_ERR(head): return PTR_ERR.
- If sysctl_perm(head, table, write ? MAY_WRITE : MAY_READ): -EPERM.
- If !table->proc_handler: -EINVAL.
- If count >= KMALLOC_MAX_SIZE: -ENOMEM.
- kbuf = kvzalloc(count+1, GFP_KERNEL); if !kbuf: -ENOMEM.
- If write: copy_from_iter_full(kbuf, count, iter); kbuf[count] = '\0'.
- error = BPF_CGROUP_RUN_PROG_SYSCTL(head, table, write, &kbuf, &count, &iocb->ki_pos); if err: free + out.
- error = table->proc_handler(table, write, kbuf, &count, &iocb->ki_pos); if err: free + out.
- If !write: copy_to_iter(kbuf, count, iter); on short: -EFAULT.
- error = count; free kbuf; head_finish; return error.

REQ-25: proc_sys_read = call_handler(write=0); proc_sys_write = call_handler(write=1).

REQ-26: proc_sys_open(inode, filp):
- head = grab_header(inode); table = PROC_I(inode)->sysctl_entry.
- If table->poll: filp->private_data = proc_sys_poll_event(table->poll).
- head_finish; return 0.

REQ-27: proc_sys_poll(filp, wait):
- head = grab_header; ret = DEFAULT_POLLMASK.
- If !table->proc_handler ∨ !table->poll: out.
- event = filp->private_data; poll_wait(filp, &table->poll->wait, wait).
- If event != atomic_read(&table->poll->event): filp->private_data = proc_sys_poll_event(table->poll); ret = EPOLLIN|EPOLLRDNORM|EPOLLERR|EPOLLPRI.
- head_finish; return ret.

REQ-28: proc_sys_poll_notify(poll):
- If !poll: return.
- atomic_inc(&poll->event); wake_up_interruptible(&poll->wait).

REQ-29: proc_sys_fill_cache(file, ctx, head, table):
- qname = {table->procname, strlen, full_name_hash}.
- child = d_lookup(dir, &qname); if !child: d_alloc_parallel + (if d_in_lookup: make_inode + d_splice_alias_ops + d_lookup_done).
- inode = d_inode(child); ino = inode->i_ino; type = inode->i_mode>>12; dput(child).
- dir_emit(ctx, qname.name, qname.len, ino, type).

REQ-30: proc_sys_link_fill_cache(file, ctx, head, table):
- head = sysctl_head_grab(head).
- If sysctl_follow_link(&head, &table) succeeds: proc_sys_fill_cache(file, ctx, head, table).
- head_finish.

REQ-31: scan(head, table, pos, file, ctx):
- (*pos)++ < ctx->pos: return true (skip already-emitted).
- If S_ISLNK(table->mode): proc_sys_link_fill_cache; else proc_sys_fill_cache.
- If emitted: ctx->pos = *pos.

REQ-32: proc_sys_readdir(file, ctx):
- head = grab_header; ctl_dir = container_of.
- dir_emit_dots.
- pos = 2; for first_entry / next_entry: scan; on stop: head_finish(h); break.
- head_finish(head); return 0.

REQ-33: proc_sys_permission(idmap, inode, mask):
- If (mask & MAY_EXEC) ∧ S_ISREG: -EACCES.
- head = grab_header; table = PROC_I(inode)->sysctl_entry.
- If !table (global root): MAY_WRITE → -EACCES, else 0.
- Else: sysctl_perm(head, table, mask & ~MAY_NOT_BLOCK).

REQ-34: proc_sys_setattr(idmap, dentry, attr):
- If (attr->ia_valid & (ATTR_MODE|ATTR_UID|ATTR_GID)): -EPERM.
- setattr_prepare; setattr_copy.

REQ-35: proc_sys_getattr(idmap, path, stat, mask, flags):
- generic_fillattr.
- If table: stat->mode = (stat->mode & S_IFMT) | table->mode.

REQ-36: proc_sys_revalidate(dir, name, dentry, flags):
- If LOOKUP_RCU: -ECHILD.
- return !PROC_I(d_inode(dentry))->sysctl->unregistering.

REQ-37: proc_sys_delete(dentry):
- return !!PROC_I(d_inode(dentry))->sysctl->unregistering.

REQ-38: proc_sys_compare(dentry, len, str, name):
- Length / memcmp first; if d_in_lookup: 0; if !inode: 1; head = READ_ONCE(...->sysctl); return !head ∨ !sysctl_is_seen(head).

REQ-39: sysctl_is_seen(p):
- spin_lock; if p->unregistering: 0; else if !set->is_seen: 1; else set->is_seen(set); spin_unlock.

REQ-40: find_subdir(dir, name, namelen):
- entry = find_entry; if !entry: -ENOENT; if !S_ISDIR(entry->mode): -ENOTDIR; return container_of(head, struct ctl_dir, header).

REQ-41: new_dir(set, name, namelen):
- kzalloc(sizeof(ctl_dir) + sizeof(ctl_node) + sizeof(ctl_table) + namelen+1, GFP_KERNEL).
- Tail-allocate: node, table, name.
- table[0].procname = copied name; table[0].mode = S_IFDIR|S_IRUGO|S_IXUGO.
- init_header(&new->header, set->dir.header.root, set, node, table, 1).

REQ-42: get_subdir(dir, name, namelen):
- spin_lock; subdir = find_subdir.
- If !IS_ERR: nreg++ found; drop_sysctl_table(&dir->header); spin_unlock; return subdir.
- If PTR_ERR != -ENOENT: fail.
- spin_unlock; new = new_dir; spin_lock.
- Re-find: maybe another CPU inserted it; if not, insert_header(dir, &new->header).
- On err: pr_err "sysctl could not get directory: <dir>/<name> <errno>".
- drop_sysctl_table(&dir->header); if new and not used: drop_sysctl_table(&new->header).

REQ-43: xlate_dir(set, dir):
- If !dir->header.parent: return &set->dir.
- parent = xlate_dir(set, dir->header.parent).
- procname = dir->header.ctl_table[0].procname; return find_subdir(parent, procname, strlen).

REQ-44: sysctl_follow_link(phead, pentry):
- spin_lock; root = (*pentry)->data; set = lookup_header_set(root); dir = xlate_dir(set, (*phead)->parent).
- If !IS_ERR(dir): find_entry in target dir; if entry ∧ use_table: unuse_table(*phead); *phead = head; *pentry = entry; ret = 0.
- spin_unlock; return ret.

REQ-45: sysctl_check_table_array(path, table):
- proc_douintvec / proc_douintvec_minmax: maxlen must == sizeof(unsigned int).
- proc_dou8vec_minmax: maxlen == sizeof(u8); extra1/2 bounded ≤ 255.
- proc_dobool: maxlen == sizeof(bool).

REQ-46: sysctl_check_table(path, header):
- For each entry: procname must be non-NULL; for value handlers (dostring/dobool/dointvec/douintvec/dointvec_minmax/dou8vec_minmax/dointvec_jiffies/...): data and maxlen non-NULL; then check_table_array.
- proc_handler must be non-NULL.
- (mode & (S_IRUGO|S_IWUGO)) must equal mode (no other bits).

REQ-47: new_links / get_links / insert_links / put_links — named-children inheritance:
- new_links(dir, head): allocate a header that contains a symlink table mirroring head; each entry has `mode = S_IFLNK|S_IRWXUGO` and `data = head->root`.
- get_links(dir, header, link_root): for every entry's procname, find an existing link in dir; either it's a real dir (matching S_ISDIR) or a link with `data == link_root`. On success, bump each link's nreg.
- insert_links(head): for non-root sets only; xlate_dir(default_set, head->parent) to find the core mirror; if get_links succeeds (reuse), 0; else new_links + drop-lock + re-lock + recheck + insert_header into core_parent.
- put_links: mirror; drop_sysctl_table on each found link.

REQ-48: sysctl_mkdir_p(dir, path):
- For each '/'-separated segment: get_subdir(dir, name, namelen); on error: break.

REQ-49: __register_sysctl_table(set, path, table, table_size):
- kzalloc(sizeof(ctl_table_header) + sizeof(ctl_node)*table_size, GFP_KERNEL_ACCOUNT).
- init_header(header, root, set, node, table, table_size).
- if sysctl_check_table(path, header): fail.
- spin_lock; dir = &set->dir; dir->header.nreg++; spin_unlock.
- dir = sysctl_mkdir_p(dir, path); if IS_ERR: fail.
- spin_lock; if insert_header(dir, header): fail_put_dir_locked.
- drop_sysctl_table(&dir->header); spin_unlock; return header.
- fail_put_dir_locked: drop_sysctl_table(&dir->header); spin_unlock.
- fail: kfree(header); return NULL.

REQ-50: register_sysctl_sz(path, table, size) = __register_sysctl_table(&sysctl_table_root.default_set, ...); EXPORT_SYMBOL.

REQ-51: register_sysctl_mount_point(path) = register_sysctl_sz(path, sysctl_mount_point, 0).

REQ-52: __register_sysctl_init(path, table, name, size):
- hdr = register_sysctl_sz; if !hdr: pr_err "failed when register_sysctl_sz %s to %s".
- kmemleak_not_leak(hdr).

REQ-53: unregister_sysctl_table(header):
- might_sleep.
- If !header: return.
- spin_lock; drop_sysctl_table(header); spin_unlock.

REQ-54: drop_sysctl_table(header):
- If --header->nreg: return.
- If parent: put_links(header); start_unregistering(header).
- If !--header->count: kfree_rcu(header, rcu).
- If parent: drop_sysctl_table(&parent->header) (cascade).

REQ-55: setup_sysctl_set(set, root, is_seen):
- memset(set, 0); set->is_seen = is_seen.
- init_header(&set->dir.header, root, set, NULL, root_table, 1).

REQ-56: retire_sysctl_set(set):
- WARN_ON(!RB_EMPTY_ROOT(&set->dir.root)) — all children must be unregistered.

REQ-57: proc_sys_init():
- proc_sys_root = proc_mkdir("sys", NULL).
- proc_sys_root->proc_iops = &proc_sys_dir_operations.
- proc_sys_root->proc_dir_ops = &proc_sys_dir_file_operations.
- proc_sys_root->nlink = 0.
- return sysctl_init_bases().

REQ-58: sysctl_aliases[] (command-line compatibility):
- `hardlockup_all_cpu_backtrace` → `kernel.hardlockup_all_cpu_backtrace`.
- `hung_task_panic` → `kernel.hung_task_panic`.
- `numa_zonelist_order` → `vm.numa_zonelist_order`.
- `softlockup_all_cpu_backtrace` → `kernel.softlockup_all_cpu_backtrace`.
- Trailing { } sentinel.

REQ-59: process_sysctl_arg(param, val, _, arg):
- If param starts with "sysctl": skip "sysctl"; require '/' or '.'; skip.
- Else: alias = sysctl_find_alias(param); if !alias: return 0.
- Require val non-empty.
- Lazy-mount procfs via kern_mount("proc") if not yet mounted (`*proc_mnt`).
- path = kasprintf("sys/%s", param); strreplace('.', '/').
- file = file_open_root_mnt(*proc_mnt, path, O_WRONLY, 0).
- kernel_write(file, val, len, &pos); on -EINVAL / short / open-err: pr_err with parameter name.
- filp_close.

REQ-60: do_sysctl_args():
- command_line = kstrdup(saved_command_line, GFP_KERNEL).
- parse_args("Setting sysctl args", command_line, NULL, 0, -1, -1, &proc_mnt, process_sysctl_arg).
- If proc_mnt: kern_unmount.

REQ-61: proc_sys_file_operations:
- open=proc_sys_open, poll=proc_sys_poll, read_iter=proc_sys_read, write_iter=proc_sys_write, splice_read=copy_splice_read, splice_write=iter_file_splice_write, llseek=default_llseek.

REQ-62: proc_sys_dir_file_operations:
- read=generic_read_dir, iterate_shared=proc_sys_readdir, llseek=generic_file_llseek.

REQ-63: proc_sys_inode_operations:
- permission=proc_sys_permission, setattr=proc_sys_setattr, getattr=proc_sys_getattr.

REQ-64: proc_sys_dir_operations:
- lookup=proc_sys_lookup, plus permission / setattr / getattr.

REQ-65: proc_sys_dentry_operations:
- d_revalidate=proc_sys_revalidate, d_delete=proc_sys_delete, d_compare=proc_sys_compare.

## Acceptance Criteria

- [ ] AC-1: `register_sysctl_sz("kernel", kern_table, N)` makes every leaf in kern_table reachable at `/proc/sys/kernel/<procname>` with the declared mode.
- [ ] AC-2: `register_sysctl_sz("net/ipv4", ipv4_table, N)` autocreates `net/` then `net/ipv4/` via `sysctl_mkdir_p`.
- [ ] AC-3: `register_sysctl_mount_point("kernel/foo")` creates a perm-empty dir; further inserts under it return -EROFS.
- [ ] AC-4: Duplicate procname under same dir: `register_sysctl_sz` returns NULL with "sysctl duplicate entry" pr_err.
- [ ] AC-5: `unregister_sysctl_table(header)` blocks until all `head->used` readers release; then `proc_sys_invalidate_dcache` purges children; only then does kfree_rcu fire.
- [ ] AC-6: A reader currently in `table->proc_handler` (use_table held) blocks an `unregister_sysctl_table` caller via the completion handshake.
- [ ] AC-7: `proc_sys_lookup` on an `S_IFLNK` entry follows the link to the current namespace's `lookup(root)` set, returning the per-netns target.
- [ ] AC-8: `proc_sys_readdir` enumerates only `use_table`-able children, in rb-tree key order (memcmp + len).
- [ ] AC-9: `proc_sys_call_handler` write of count ≥ KMALLOC_MAX_SIZE: -ENOMEM (no allocation attempt).
- [ ] AC-10: `proc_sys_call_handler` write: kbuf NUL-terminates at `kbuf[count]`; on copy_from_iter_full failure: -EFAULT, no proc_handler call.
- [ ] AC-11: `proc_sys_permission`: non-root, non-root-group, mode 0644: write returns -EACCES; read returns 0.
- [ ] AC-12: `proc_sys_permission`: `mask & MAY_EXEC` on S_ISREG: -EACCES regardless of mode.
- [ ] AC-13: `proc_sys_setattr` rejects `ATTR_MODE | ATTR_UID | ATTR_GID` with -EPERM (chmod/chown of /proc/sys files forbidden).
- [ ] AC-14: `proc_sys_poll` returns `EPOLLIN|EPOLLRDNORM|EPOLLERR|EPOLLPRI` after `proc_sys_poll_notify`; idle returns `DEFAULT_POLLMASK`.
- [ ] AC-15: `sysctl_check_table` rejects: missing procname, missing proc_handler, missing data/maxlen for value-handlers, bogus mode bits.
- [ ] AC-16: `sysctl_check_table_array` rejects: proc_douintvec with maxlen ≠ sizeof(unsigned int); proc_dou8vec_minmax with maxlen ≠ sizeof(u8) or extra1/2 > 255; proc_dobool with maxlen ≠ sizeof(bool).
- [ ] AC-17: `do_sysctl_args` with `sysctl./kernel/sysrq=1` on cmdline: writes "1" to `/proc/sys/kernel/sysrq` after lazy `kern_mount("proc")`.
- [ ] AC-18: `do_sysctl_args` with `numa_zonelist_order=node` on cmdline: resolves alias to `vm.numa_zonelist_order` and writes.
- [ ] AC-19: Per-netns registration of `net/ipv4/ip_forward` in netns A is not visible from netns B's `is_seen` predicate.
- [ ] AC-20: `proc_sys_dentry_operations.d_compare` returns non-zero for an unregistering or unseen sysctl, forcing dcache miss.

## Architecture

```rust
struct CtlTable {
    procname: &'static CStr,
    data: *mut u8,
    maxlen: usize,
    mode: u16,                          // S_IF* | UNIX bits
    table_type: CtlTableType,           // Default | PermanentlyEmpty
    proc_handler: Option<ProcHandlerFn>,
    extra1: *const u8,
    extra2: *const u8,
    poll: Option<*const CtlTablePoll>,
}

type ProcHandlerFn = extern "C" fn(
    table: &CtlTable, write: bool, buffer: *mut u8, lenp: &mut usize, ppos: &mut i64
) -> i32;

struct CtlTableHeader {
    ctl_table: *const [CtlTable],
    ctl_table_size: usize,
    ctl_table_arg: *const CtlTable,
    used: u32,
    count: u32,
    nreg: u32,
    unregistering: AtomicPtr<Completion>,    // NULL | &completion | ERR_PTR(-EINVAL)
    root: *const CtlTableRoot,
    set: *const CtlTableSet,
    parent: Option<*const CtlDir>,
    node: *mut [CtlNode],                    // tail-allocated, one per ctl_table entry
    inodes: HlistHead,                       // sibling_inodes for invalidate_dcache
    rcu: RcuHead,
}

struct CtlNode {
    node: RbNode,
    header: *const CtlTableHeader,
}

struct CtlDir {
    header: CtlTableHeader,                  // first field; container_of relies on this
    root: RbRoot,
}

struct CtlTableSet {
    dir: CtlDir,
    is_seen: Option<fn(&CtlTableSet) -> bool>,
}

struct CtlTableRoot {
    default_set: CtlTableSet,
    lookup: Option<fn(&CtlTableRoot) -> &CtlTableSet>,
    permissions: Option<fn(&CtlTableHeader, &CtlTable) -> u16>,
    set_ownership: Option<fn(&CtlTableHeader, &mut Kuid, &mut Kgid)>,
}

static SYSCTL_LOCK: SpinLock<()> = SpinLock::new(());
static SYSCTL_TABLE_ROOT: CtlTableRoot = /* synthetic root_table */;
```

`procfs::sysctl::register_table_inner(set, path, table) -> Option<*CtlTableHeader>`:
1. header = kzalloc(sizeof(CtlTableHeader) + sizeof(CtlNode)*N).
2. init_header(header, set.root, set, node, table, N).
3. If sysctl_check_table(path, header): kfree + None.
4. SYSCTL_LOCK; dir = &set.dir; dir.header.nreg++; unlock.
5. dir = sysctl_mkdir_p(dir, path); on err: kfree + None.
6. SYSCTL_LOCK; if insert_header(dir, header): drop_sysctl_table(&dir.header); unlock; kfree + None.
7. drop_sysctl_table(&dir.header); unlock.
8. Some(header).

`procfs::sysctl::unregister_table(header)`:
1. might_sleep.
2. If header.is_null: return.
3. SYSCTL_LOCK; drop_sysctl_table(header); unlock.

`procfs::sysctl::drop_table(header)`:
1. If --header.nreg > 0: return.
2. If header.parent.is_some(): put_links(header); start_unregistering(header).
3. If --header.count == 0: kfree_rcu(header).
4. If header.parent.is_some(): drop_table(parent).

`procfs::sysctl::start_unregistering(p)`:
1. lockdep_assert SYSCTL_LOCK held.
2. If p.used > 0:
   - completion = Completion::new(); p.unregistering = &completion.
   - SYSCTL_LOCK.unlock(); wait_for_completion(&completion).
3. Else: p.unregistering = ERR_PTR(-EINVAL); SYSCTL_LOCK.unlock().
4. proc_invalidate_siblings_dcache(&p.inodes, &SYSCTL_LOCK).
5. SYSCTL_LOCK.lock(); erase_header(p).

`procfs::sysctl::vfs_lookup(dir, dentry, flags)`:
1. head = grab_header(dir); err on ERR_PTR.
2. ctl_dir = container_of(head, CtlDir, header).
3. p = lookup_entry(&h, ctl_dir, name); if !p: out.
4. If S_ISLNK(p.mode): sysctl_follow_link(&h, &p).
5. inode = make_inode(sb, h ?: head, p).
6. d_splice_alias_ops(inode, dentry, &DENTRY_OPS).
7. head_finish(h); head_finish(head).

`procfs::sysctl::call_handler(iocb, iter, write)`:
1. inode = file_inode; head = grab_header; table = PROC_I(inode).sysctl_entry.
2. permission check via sysctl_perm.
3. table.proc_handler required, count < KMALLOC_MAX_SIZE.
4. kbuf = kvzalloc(count+1).
5. If write: copy_from_iter_full + NUL.
6. BPF_CGROUP_RUN_PROG_SYSCTL hook.
7. table.proc_handler(table, write, kbuf, &count, &iocb.ki_pos).
8. If !write: copy_to_iter.
9. kvfree(kbuf); head_finish(head); return count or error.

`procfs::sysctl::readdir(file, ctx)`:
1. head = grab_header; ctl_dir.
2. dir_emit_dots.
3. pos = 2; for first_entry / next_entry: scan; on stop: head_finish + break.
4. head_finish(head); return 0.

`procfs::sysctl::follow_link(phead, pentry)`:
1. SYSCTL_LOCK.
2. root = (*pentry).data as *CtlTableRoot.
3. set = lookup_header_set(root).
4. dir = xlate_dir(set, (*phead).parent).
5. find_entry(&head, dir, procname); if use_table: unuse_table(*phead); *phead = head; *pentry = entry.
6. SYSCTL_LOCK.unlock.

`procfs::sysctl::sysctl_mkdir_p(dir, path)`:
1. For each '/' segment of path:
   - dir = get_subdir(dir, name, namelen); on err: break.
2. return dir.

`procfs::sysctl::get_subdir(dir, name, namelen)`:
1. SYSCTL_LOCK; subdir = find_subdir(dir, name, namelen).
2. If Ok(subdir): nreg++; drop_sysctl_table(&dir.header); unlock; return subdir.
3. If err != -ENOENT: fail.
4. unlock; new = new_dir(set, name, namelen); SYSCTL_LOCK.
5. Re-find: if exists, found; else insert_header(dir, &new.header).
6. drop_sysctl_table(&dir.header); if new not used: drop_sysctl_table(&new.header).
7. unlock; return subdir.

`procfs::sysctl::insert_links / put_links / new_links / get_links` — named-children inheritance:
- Only for non-default sets.
- xlate_dir(default_set, head.parent) finds the corresponding directory in the root mount-namespace's set.
- If existing links match (S_ISDIR-on-S_ISDIR or S_ISLNK with .data == head.root): reuse via nreg++.
- Else: allocate a link-header (each entry S_IFLNK|S_IRWXUGO; .data = head.root); insert_header(core_parent, links).

`procfs::sysctl::init`:
1. proc_sys_root = proc_mkdir("sys", NULL).
2. proc_sys_root.proc_iops = &IOPS_DIR.
3. proc_sys_root.proc_dir_ops = &FOPS_DIR.
4. proc_sys_root.nlink = 0.
5. sysctl_init_bases().

`procfs::sysctl::cmdline_args` — process_sysctl_arg + do_sysctl_args:
1. parse `saved_command_line` for `sysctl./path=value` or aliased `<alias>=value`.
2. Lazy `kern_mount("proc")`; path = "sys/<param-with-./-translated-to-/>".
3. file = file_open_root_mnt(O_WRONLY); kernel_write(file, val, len, &pos).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sysctl_lock_held_for_mutators` | INVARIANT | per-insert_entry / erase_entry / use_table / unuse_table / find_entry: SYSCTL_LOCK held. |
| `use_count_balanced` | INVARIANT | per-readside: every use_table paired with unuse_table on every exit path. |
| `nreg_drops_to_zero` | INVARIANT | per-drop_sysctl_table: nreg-- ≥ 0; on hitting 0 the parent cascade fires exactly once. |
| `count_drops_to_zero` | INVARIANT | per-drop_sysctl_table: --header.count ≥ 0; on 0, kfree_rcu fires exactly once. |
| `unregistering_completion_handshake` | INVARIANT | per-start_unregistering: if used > 0, the unique completion is signaled by the last unuse_table. |
| `dcache_invalidated_before_free` | INVARIANT | per-start_unregistering: proc_sys_invalidate_dcache runs before erase_header. |
| `kbuf_size_bounded` | INVARIANT | per-call_handler: count < KMALLOC_MAX_SIZE; kbuf NUL-terminated at kbuf[count]. |
| `perm_empty_no_children` | INVARIANT | per-insert_header: perm-empty dir rejects further inserts (-EROFS). |
| `make_inode_count_balanced` | INVARIANT | per-make_inode: head.count++; per-evict_inode: --head.count with kfree_rcu on 0. |
| `S_IFLNK_data_is_root` | INVARIANT | per-new_links: every link's .data points at the originating ctl_table_root. |
| `rb_tree_key_uniqueness` | INVARIANT | per-insert_entry: namecmp duplicates return -EEXIST; no rb-insert of duplicate keys. |

### Layer 2: TLA+

`fs/proc/proc-sysctl-tree.tla`:
- States per registration: alloc → init_header → check_table → mkdir_p → insert_header → reachable.
- States per unregistration: nreg-- → put_links → start_unregistering (drain) → erase_header → cascade-to-parent → kfree_rcu.
- States per readside: vfs_lookup → grab_header → lookup_entry → follow_link? → make_inode → use_table-held → ... → head_finish → unuse_table.
- States per writeside: call_handler → grab_header → sysctl_perm → kvzalloc → copy_from_iter → bpf_cgroup_hook → proc_handler → kvfree → head_finish.
- Properties:
  - `safety_no_use_after_unregister` — per-handler: a successfully grabbed head cannot be freed before head_finish.
  - `safety_dcache_purge_precedes_free` — per-unregister: dcache invalidation precedes RCU-free.
  - `safety_no_duplicate_procname` — per-dir: rb-tree key uniqueness for every (parent, procname).
  - `safety_perm_empty_immutable` — per-perm-empty dir: no descendant insert succeeds.
  - `safety_lookup_namespace_isolation` — per-`set->is_seen` returning false: lookup/readdir cannot expose the table.
  - `liveness_register_completes` — per-register_sysctl_sz: alloc + insert ⟹ returns header (modulo OOM).
  - `liveness_unregister_drains` — per-unregister: completes after the last reader's unuse_table.
  - `liveness_reader_terminates` — per-call_handler: any reader either gets ERR_PTR(-ENOENT) immediately or completes after at most one proc_handler invocation.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `register_table_inner` post: Some(h) ⟹ h.nreg==1 ∧ h reachable from set.dir.root via rb-tree; None ⟹ no kalloc leak | `procfs::sysctl::register_table_inner` |
| `unregister_table` post: header unreachable from any set.dir.root; head.count → 0 ⟹ kfree_rcu | `procfs::sysctl::unregister_table` |
| `drop_table` post: header.nreg == old-1; cascade fires iff transitioning 1→0 | `procfs::sysctl::drop_table` |
| `start_unregistering` post: p.unregistering != NULL; p.used == 0 on return; dcache invalidated | `procfs::sysctl::start_unregistering` |
| `vfs_lookup` post: returns ERR_PTR(-ENOENT) for missing or unregistering entry; otherwise returns dentry pointing at a fresh inode | `procfs::sysctl::vfs_lookup` |
| `call_handler` post: returns count on success, -EPERM / -EINVAL / -ENOMEM / -EFAULT / proc_handler-err per branch; kbuf freed on every exit | `procfs::sysctl::call_handler` |
| `readdir` post: emits every use_table-able child exactly once per ctx position; head_finish called on every exit | `procfs::sysctl::readdir` |
| `follow_link` post: *phead, *pentry point at target dir entry in current namespace's set; -ENOENT if target gone | `procfs::sysctl::follow_link` |
| `insert_links` post: namespaced header has matching symlinks under default_set's mirror dir | `procfs::sysctl::insert_links` |
| `put_links` post: every namespaced header's mirror links are nreg-- once | `procfs::sysctl::put_links` |
| `check_table` post: all listed validation rules met or -EINVAL | `procfs::sysctl::check_table` |
| `perm` post: equivalent to `test_perm(effective_mode, op)` after root override | `procfs::sysctl::perm` |

### Layer 4: Verus/Creusot functional

`Per-/proc/sys/<path>` semantic equivalence with reference golden corpus:
- procps `sysctl -a` listing: every documented entry under kernel/, vm/, net/, fs/, abi/, debug/, dev/, user/ appears with the upstream-documented name and mode.
- `sysctl -p /etc/sysctl.conf` writes: each line `key = value` resolves through `process_sysctl_arg` (or VFS write) to the same kernel state mutation.
- Per-`Documentation/admin-guide/sysctl/*.rst` entries: every key documented appears at the documented path with the documented type.
- Per-netns isolation: `net.ipv4.ip_forward` is per-netns settable; `kernel.printk` is global.
- Per-`sysctl_aliases[]`: cmdline params `hardlockup_all_cpu_backtrace`, `hung_task_panic`, `numa_zonelist_order`, `softlockup_all_cpu_backtrace` resolve to `kernel.*` / `vm.*` paths.

## Hardening

(Inherits row-1 features from `fs/proc/00-overview.md` § Hardening.)

proc_sysctl reinforcement:

- **Per-spinlock-protected rb-tree mutations** — defense against per-concurrent-insert / per-concurrent-erase races.
- **Per-`use_table` / `unuse_table` reader-count** — defense against per-unregister-while-reading UAF.
- **Per-completion handshake in `start_unregistering`** — defense against per-incomplete-drain ⟹ proc_handler called on freed table.
- **Per-`proc_sys_invalidate_dcache` before erase** — defense against per-stale-dentry returning freed inode.
- **Per-RCU-freed header (`kfree_rcu`)** — defense against per-walker-in-RCU-section-sees-freed-memory.
- **Per-`unregistering` sentinel checked in d_revalidate / d_compare** — defense against per-cached-dentry resurrection.
- **Per-`sysctl_check_table` at register time** — defense against per-malformed-table (NULL procname / handler / data, bogus mode bits).
- **Per-`sysctl_check_table_array` size enforcement** — defense against per-overflow-write into `data[maxlen..]`.
- **Per-perm-empty marker (`SYSCTL_TABLE_TYPE_PERMANENTLY_EMPTY`)** — defense against per-unprivileged-mount filling a known-empty mountpoint.
- **Per-`KMALLOC_MAX_SIZE` write cap** — defense against per-large-allocation DoS via `/proc/sys/.../foo` write.
- **Per-`kvzalloc(count+1)` + explicit NUL** — defense against per-non-terminated-string handler.
- **Per-`copy_from_iter_full` strict count match** — defense against per-short-copy ⟹ proc_handler reads stale tail.
- **Per-`sysctl_perm` (no root-blanket)** — defense against per-root-bypass of read-only sysctls (some are read-only even to root).
- **Per-S_IFEXEC denial on regular sysctls** — defense against per-`mmap`-as-executable of a tunable.
- **Per-`ATTR_MODE | ATTR_UID | ATTR_GID` denial in setattr** — defense against per-chmod / chown of /proc/sys files.
- **Per-`BPF_CGROUP_RUN_PROG_SYSCTL` hook before proc_handler** — defense via cgroup policy on per-container write.
- **Per-`is_seen` namespace gate** — defense against per-cross-netns leak.
- **Per-`namecmp` deterministic ordering** — defense against per-rb-tree ambiguity from prefix collisions (memcmp + len-tiebreak).
- **Per-`xlate_dir` translation for link tree** — defense against per-dangling symlink to gone-away namespace.
- **Per-`kmemleak_not_leak` on __register_sysctl_init** — defense against per-false-leak-report for permanently-resident init tables.

## Open Questions

- Per-`ctl_table_poll` lifetime: the upstream code requires the poll structure to outlive any open file descriptor referencing it; Rookery should model this as a borrow-checked lifetime in `CtlTable`.
- Per-`bpf-cgroup` hook integration timing: `BPF_CGROUP_RUN_PROG_SYSCTL` may mutate kbuf / count / ki_pos. The Rookery contract for BPF cgroup integration is captured in the bpf-cgroup Tier-3; cross-link required.
- Per-`SYSCTL_TABLE_TYPE_*` enum extensions: upstream introduced perm-empty for unprivileged-mount safety; future types should be additive.
- Per-`__init` allocation accounting: `GFP_KERNEL_ACCOUNT` for headers means cgroup memcg accounting includes sysctl tables; should match upstream.

## Out of Scope

- Per-handler implementations (`proc_dointvec`, `proc_dostring`, `proc_dointvec_minmax`, `proc_doulongvec_minmax`, `proc_dou8vec_minmax`, `proc_dobool`, `proc_dointvec_jiffies` / `_userhz_jiffies` / `_ms_jiffies`, `proc_douintvec` / `_minmax`, `proc_doulongvec_ms_jiffies_minmax`) — these live in `kernel/sysctl.c` and are covered in the existing `proc-sysctl.md` Tier-3 design and in `kernel-sysctl.md` Tier-3 (per-handler).
- Per-`sysctl_init_bases()` registration of the canonical kernel/, vm/, fs/, debug/, dev/, abi/, user/ subtrees — covered in `kernel-sysctl.md` Tier-3.
- BPF cgroup sysctl integration internals (`BPF_CGROUP_RUN_PROG_SYSCTL` callback shape) — covered in `bpf-cgroup.md` Tier-3.
- Per-netns sysctl set construction (`net/sysctl_net.c` net namespace sysctl_table_root) — covered in `sysctl-net.md` Tier-3.
- `proc-sysctl.md` (existing Tier-3): per-handler family and the user-visible /proc/sys/* compatibility surface (paths like `net.ipv4.ip_forward`, `kernel.randomize_va_space`, `vm.swappiness`, etc.); this document is the structural / mechanism side, that document is the per-key wire-format side.
- Implementation code.
