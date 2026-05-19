# Tier-3: kernel/cgroup/cgroup-v1.c — Legacy cgroup-v1 hierarchies, mount/remount, ABI surface

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/cgroup/00-overview.md
upstream-paths:
  - kernel/cgroup/cgroup-v1.c (~1372 lines)
  - kernel/cgroup/cgroup-internal.h
  - include/linux/cgroup.h
  - include/uapi/linux/cgroupstats.h
  - Documentation/admin-guide/cgroup-v1/*
-->

## Summary

cgroup-v1 supports **multiple named hierarchies**, each containing a disjoint subset of controllers, mounted independently. Per-hierarchy: one `struct cgroup_root` with a name + subsys_mask. Per-mount: a parser (`cgroup1_fs_parameters`) accepts comma-separated controller names (`cpu,cpuacct,memory,...`), the special tokens `all` / `none` / `name=<id>`, plus flags `noprefix`, `xattr`, `cpuset_v2_mode`, `clone_children`, `release_agent=<path>`, `favordynmods` / `nofavordynmods`. Per-`cgroup1_get_tree` resolves (subsys-mask, name) tuple against existing roots; if a match exists with identical mask it is reused, otherwise a new `cgroup_root` is allocated and `cgroup_setup_root` plumbs it. Per-`cgroup1_reconfigure` allows remount that rebinds added/removed controllers but rejects flag/name changes and refuses to remount populated hierarchies. Per-ABI: `tasks` + `cgroup.procs` (pidlist), `cgroup.clone_children`, `cgroup.sane_behavior`, `notify_on_release`, `release_agent` (root-only), plus controller-private files. Per-`release_agent` invocation: on transition to "no tasks and no children", queue `release_agent_work` which calls `call_usermodehelper(agentbuf, [agentbuf, cgrp-path], envp, UMH_WAIT_EXEC)` so userspace can rmdir the cgroup. Per-`cgroup_attach_task_all` migrates a task to the same cgroup as another task across *every* legacy hierarchy. Per-`proc_cgroupstats_show` (`/proc/cgroups`) enumerates legacy controllers + hierarchy IDs. Per-`task_get_cgroup1` resolves (task, hierarchy_id) → cgroup for BPF / introspection. Critical for: backwards compatibility with systemd-pre-unified-hierarchy, Android, container runtimes pinned to v1 cgroups, hybrid v1+v2 deployments.

This Tier-3 covers `kernel/cgroup/cgroup-v1.c` (~1372 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cgroup_pidlist` | per-pid-namespace pidlist for tasks/procs file | `CgroupV1::Pidlist` |
| `cgroup1_base_files[]` | per-hierarchy ABI files (tasks, procs, release_agent, ...) | `CgroupV1::BASE_FILES` |
| `cgroup1_fs_parameters[]` | per-mount option spec | `CgroupV1::FS_PARAMETERS` |
| `cgroup1_parse_param()` | per-fs-param parse | `CgroupV1::parse_param` |
| `check_cgroupfs_options()` | per-mount option validation | `CgroupV1::check_options` |
| `cgroup1_root_to_use()` | per-mount root reuse / create | `CgroupV1::root_to_use` |
| `cgroup1_get_tree()` | per-mount entry | `CgroupV1::get_tree` |
| `cgroup1_reconfigure()` | per-remount | `CgroupV1::reconfigure` |
| `cgroup1_rename()` | per-rmdir-rename of dirs in place | `CgroupV1::rename` |
| `cgroup1_show_options()` | per-/proc/mounts options | `CgroupV1::show_options` |
| `cgroup1_kf_syscall_ops` | per-kernfs syscall vtable | `CgroupV1::KF_SYSCALL_OPS` |
| `__cgroup1_procs_write()` | per-write to tasks / cgroup.procs | `CgroupV1::procs_write_inner` |
| `cgroup1_procs_write` / `cgroup1_tasks_write` | per-procs / per-task write | `CgroupV1::procs_write` / `tasks_write` |
| `cgroup_pidlist_start/next/stop/show` | per-seq-file iterator over pidlist | `CgroupV1::Pidlist::iter*` |
| `pidlist_array_load()` | per-snapshot pidarray | `CgroupV1::pidlist_array_load` |
| `pidlist_uniq()` | per-dedup of sorted pid array | `CgroupV1::pidlist_uniq` |
| `cgroup_pidlist_destroy_work_fn` | per-delayed destruction | `CgroupV1::pidlist_destroy_work` |
| `cgroup1_pidlist_destroy_all()` | per-cgroup-destroy pidlist flush | `CgroupV1::pidlist_destroy_all` |
| `cgroup_release_agent_write/show` | per-release_agent file | `CgroupV1::release_agent_rw` |
| `cgroup1_release_agent()` | per-work invocation of release agent | `CgroupV1::release_agent_work` |
| `cgroup1_check_for_release()` | per-empty cgroup release trigger | `CgroupV1::check_for_release` |
| `cgroup_sane_behavior_show()` | per-sane_behavior compat | `CgroupV1::sane_behavior_show` |
| `cgroup_read_notify_on_release` / `_write_*` | per-notify_on_release flag | `CgroupV1::notify_on_release_rw` |
| `cgroup_clone_children_read` / `_write` | per-cpuset CLONE_CHILDREN flag | `CgroupV1::clone_children_rw` |
| `cgroup_attach_task_all()` | per-task migrate-to-all-v1-cgroups | `CgroupV1::attach_task_all` |
| `cgroup_transfer_tasks()` | per-cgroup task migration drain | `CgroupV1::transfer_tasks` |
| `proc_cgroupstats_show()` | per-/proc/cgroups | `CgroupV1::proc_cgroupstats_show` |
| `cgroupstats_build()` | per-taskstats cgroupstats export | `CgroupV1::cgroupstats_build` |
| `task_get_cgroup1()` | per-(task, hierarchy_id) lookup | `CgroupV1::task_get_cgroup1` |
| `cgroup1_ssid_disabled()` | per-cgroup_no_v1= disable check | `CgroupV1::ssid_disabled` |
| `cgroup_no_v1=` / `cgroup_v1_proc=` boot params | per-cmdline gate | shared |

## Compatibility contract

REQ-1: struct cgroup_pidlist:
- key.type: CGROUP_FILE_PROCS or CGROUP_FILE_TASKS.
- key.ns: pid_namespace (different namespaces see different pids ⟹ different lists).
- list: pid_t array (sorted, deduplicated).
- length: number of valid entries.
- links: list_head into cgroup->pidlists.
- owner: back-pointer to cgroup.
- destroy_dwork: delayed_work used to lazily free after `CGROUP_PIDLIST_DESTROY_DELAY` (1 HZ).

REQ-2: cgroup1_base_files[]:
- "cgroup.procs": seq_show via pidlist iter, write via cgroup1_procs_write (threadgroup=true).
- "cgroup.clone_children": u64 read/write (CGRP_CPUSET_CLONE_CHILDREN bit).
- "cgroup.sane_behavior": ONLY_ON_ROOT, returns "0\n" (deprecated v1 compat).
- "tasks": pidlist iter, write via cgroup1_tasks_write (threadgroup=false).
- "notify_on_release": u64 read/write (CGRP_NOTIFY_ON_RELEASE).
- "release_agent": ONLY_ON_ROOT; read via cgroup_release_agent_show, write via cgroup_release_agent_write, max_write_len = PATH_MAX - 1.
- Terminator `{}`.

REQ-3: cgroup1_fs_parameters:
- "all": flag (Opt_all) — implicit "all enabled v1 controllers".
- "clone_children": flag (Opt_clone_children).
- "cpuset_v2_mode": flag (Opt_cpuset_v2_mode).
- "name": string (Opt_name) — named hierarchy id, matches `[\w.-]+`, max MAX_CGROUP_ROOT_NAMELEN-1.
- "none": flag (Opt_none) — explicitly no controllers (named hierarchy).
- "noprefix": flag (Opt_noprefix) — strip "cpuset." prefix on cpuset-only hierarchies.
- "release_agent": string (Opt_release_agent) — requires CAP_SYS_ADMIN + init user_ns.
- "xattr": flag (Opt_xattr).
- "favordynmods" / "nofavordynmods": flag pair (Opt_favordynmods / Opt_nofavordynmods).
- Anything else: treated by cgroup1_parse_param as a controller `legacy_name`; OR's `1 << ssid` into ctx.subsys_mask if enabled and not blocked by `cgroup_no_v1=`.

REQ-4: cgroup1_parse_param(fc, param):
- fs_parse → result.
- If -ENOPARAM: try vfs_parse_fs_param_source(fc, param); if still -ENOPARAM, walk for_each_subsys(ss, i):
  - if !strcmp(param->key, ss->legacy_name) ∧ !cgroup1_subsys_absent(ss):
    - if !cgroup_ssid_enabled(i) ∨ cgroup1_ssid_disabled(i): invalfc("Disabled controller").
    - ctx.subsys_mask |= 1 << i; return 0.
  - else: invalfc("Unknown subsys name").
- Per-opt switch:
  - Opt_none: ctx.none = true.
  - Opt_all: ctx.all_ss = true.
  - Opt_noprefix: ctx.flags |= CGRP_ROOT_NOPREFIX.
  - Opt_clone_children: ctx.cpuset_clone_children = true.
  - Opt_cpuset_v2_mode: ctx.flags |= CGRP_ROOT_CPUSET_V2_MODE.
  - Opt_xattr: ctx.flags |= CGRP_ROOT_XATTR.
  - Opt_favordynmods: ctx.flags |= CGRP_ROOT_FAVOR_DYNMODS.
  - Opt_nofavordynmods: ctx.flags &= ~CGRP_ROOT_FAVOR_DYNMODS.
  - Opt_release_agent: forbid re-spec; require init user_ns + CAP_SYS_ADMIN; consume param->string.
  - Opt_name: forbid if cgroup_no_v1_named; require non-empty + len <= MAX_CGROUP_ROOT_NAMELEN-1 + match `[\w.-]+`; forbid re-spec; consume param->string.

REQ-5: check_cgroupfs_options(fc):
- mask = U32_MAX; if CONFIG_CPUSETS: mask = ~(1 << cpuset_cgrp_id).
- enabled = subsystems where cgroup_ssid_enabled ∧ !cgroup1_ssid_disabled ∧ !cgroup1_subsys_absent.
- ctx.subsys_mask &= enabled.
- If subsys_mask == 0 ∧ !ctx.none ∧ !ctx.name: ctx.all_ss = true.
- If ctx.all_ss:
  - If subsys_mask != 0: invalfc("subsys name conflicts with all").
  - subsys_mask = enabled.
- If subsys_mask == 0 ∧ !ctx.name: invalfc("Need name or subsystem set").
- If (flags & CGRP_ROOT_NOPREFIX) ∧ (subsys_mask & mask): invalfc("noprefix used incorrectly").
- If subsys_mask ∧ ctx.none: invalfc("none used incorrectly").

REQ-6: cgroup1_root_to_use(fc):
- check_cgroupfs_options(fc).
- For each requested ss != dfl: drain dying root via percpu_ref_tryget_live; on fail return 1 (restart).
- For each existing root != cgrp_dfl_root:
  - If ctx.name and root->name mismatch: continue.
  - If (ctx.subsys_mask ∨ ctx.none) ∧ ctx.subsys_mask != root->subsys_mask: continue if no name match, else -EBUSY.
  - If root->flags ^ ctx.flags: warn but ignore.
  - ctx.root = root; return 0.
- Else create new: !subsys_mask ∧ !ctx.none ⟹ invalfc; ctx.ns != init_cgroup_ns ⟹ -EPERM.
- kzalloc cgroup_root; init_cgroup_root(ctx); cgroup_setup_root(root, ctx.subsys_mask); favor_dynmods on success; free on err.

REQ-7: cgroup1_get_tree(fc):
- Require ns_capable(ctx.ns->user_ns, CAP_SYS_ADMIN); else -EPERM.
- cgroup_lock_and_drain_offline(&cgrp_dfl_root.cgrp).
- cgroup1_root_to_use(fc); if !ret ∧ !percpu_ref_tryget_live(&ctx.root->cgrp.self.refcnt): ret = 1.
- cgroup_unlock.
- If !ret: cgroup_do_get_tree(fc).
- If !ret ∧ percpu_ref_is_dying: fc_drop_locked(fc); ret = 1.
- If ret > 0: msleep(10); restart_syscall().

REQ-8: cgroup1_reconfigure(fc):
- cgroup_lock_and_drain_offline.
- check_cgroupfs_options.
- If subsys_mask change OR release_agent set: pr_warn deprecation.
- added_mask = ctx.subsys_mask & ~root->subsys_mask; removed_mask = root->subsys_mask & ~ctx.subsys_mask.
- If (flags ^ root->flags) OR (name && strcmp(name, root->name)): errorfc + -EINVAL.
- If root has children: -EBUSY.
- rebind_subsystems(root, added_mask); WARN_ON(rebind_subsystems(&cgrp_dfl_root, removed_mask)).
- If ctx.release_agent: strscpy under release_agent_path_lock.
- trace_cgroup_remount.

REQ-9: cgroup1_show_options(seq, kf_root):
- For each ss in root->subsys_mask: seq_show_option(legacy_name).
- Per-flag: NOPREFIX → ",noprefix"; XATTR → ",xattr"; CPUSET_V2_MODE → ",cpuset_v2_mode"; FAVOR_DYNMODS → ",favordynmods".
- Under release_agent_path_lock: if strlen(release_agent_path) > 0: seq_show_option("release_agent", path).
- If CGRP_CPUSET_CLONE_CHILDREN on root cgrp: ",clone_children".
- If root->name set: seq_show_option("name", root->name).

REQ-10: __cgroup1_procs_write(of, buf, nbytes, off, threadgroup):
- cgrp = cgroup_kn_lock_live(of->kn, false); else -ENODEV.
- task = cgroup_procs_write_start(buf, threadgroup, &lock_mode); ret = PTR_ERR_OR_ZERO(task).
- Capability check (uses f_cred to defend against fd-inheritance escalation):
  - cred = of->file->f_cred; tcred = get_task_cred(task).
  - if !root && cred.euid != tcred.uid && cred.euid != tcred.suid: -EACCES.
- cgroup_attach_task(cgrp, task, threadgroup).
- cgroup_procs_write_finish(task, lock_mode); cgroup_kn_unlock(of->kn).

REQ-11: Pidlist file iteration:
- start(): under cgrp->pidlist_mutex, find existing list for (filetype, pid_ns) via cgroup_pidlist_find; else pidlist_array_load + create; binary-search to *pos; return iter pointer.
- next(): advance pointer; return NULL at end.
- stop(): mod_delayed_work(destroy_dwork, CGROUP_PIDLIST_DESTROY_DELAY = 1 HZ); release pidlist_mutex.
- show(): seq_printf("%d\n", *iter).

REQ-12: pidlist_array_load(cgrp, type, *lp):
- length = cgroup_task_count(cgrp).
- array = kvmalloc(pid_t[length]).
- css_task_iter walk; for each task: pid = task_tgid_vnr (PROCS) or task_pid_vnr (TASKS); skip pid <= 0.
- sort(cmppid), then pidlist_uniq.
- cgroup_pidlist_find_create; replace l->list (kvfree old); l->length = length.

REQ-13: pidlist_uniq(list, length): in-place dedup of sorted array; O(n).

REQ-14: cgroup1_pidlist_destroy_all(cgrp):
- Under pidlist_mutex: mod_delayed_work(each l->destroy_dwork, 0) (drain).
- flush_workqueue(cgroup_pidlist_destroy_wq).
- BUG_ON(!list_empty(&cgrp->pidlists)).

REQ-15: cgroup_release_agent_write:
- BUILD_BUG_ON(sizeof(root->release_agent_path) < PATH_MAX).
- Require ctx->ns->user_ns == init_user_ns ∧ file_ns_capable(of->file, init_user_ns, CAP_SYS_ADMIN); else -EPERM.
- Under release_agent_path_lock: strscpy(root->release_agent_path, strstrip(buf), sizeof).

REQ-16: cgroup1_release_agent(work):
- cgrp = container_of(work, cgroup, release_agent_work).
- If empty path: return.
- kmalloc pathbuf + agentbuf (PATH_MAX each).
- Under release_agent_path_lock: strscpy(agentbuf, root->release_agent_path).
- cgroup_path_ns(cgrp, pathbuf, PATH_MAX, &init_cgroup_ns).
- argv = [agentbuf, pathbuf, NULL]; envp = ["HOME=/", "PATH=/sbin:/bin:/usr/sbin:/usr/bin", NULL].
- call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC).

REQ-17: cgroup1_check_for_release(cgrp):
- If notify_on_release(cgrp) ∧ !cgroup_is_populated(cgrp) ∧ !css_has_online_children(&cgrp->self) ∧ !cgroup_is_dead(cgrp):
  - schedule_work(&cgrp->release_agent_work).

REQ-18: cgroup1_rename(kn, new_parent, new_name_str):
- Reject '\n' in name (would corrupt /proc/<pid>/cgroup).
- Require KERNFS_DIR.
- Require rcu_access_pointer(kn->__parent) == new_parent (no cross-parent rename).
- Break kernfs active_ref protection (new_parent + kn); cgroup_lock; kernfs_rename; trace_cgroup_path on success.

REQ-19: Boot params:
- `cgroup_no_v1=<list>`: comma-separated controller names + special "all" / "named"; populates `cgroup_no_v1_mask` (per-ssid block) + `cgroup_no_v1_named` (forbid name= mounts).
- `cgroup_v1_proc=<bool>`: `proc_show_all` — when set, `/proc/cgroups` enumerates legacy controllers even if no v1 hierarchy mounted.

## Acceptance Criteria

- [ ] AC-1: `mount -t cgroup -o memory,cpu none /sys/fs/cgroup/test`: parser accepts; new cgroup_root with subsys_mask={memory,cpu}.
- [ ] AC-2: Second mount with same mask + same name: ctx.root reused (no new root).
- [ ] AC-3: Second mount with same name + different subsys_mask: -EBUSY.
- [ ] AC-4: Mount with `name=foo,none`: named hierarchy with zero controllers succeeds.
- [ ] AC-5: Mount with `noprefix` + non-cpuset controller: -EINVAL ("noprefix used incorrectly").
- [ ] AC-6: Mount with `name=...` containing `\n` or invalid char: -EINVAL.
- [ ] AC-7: Remount that changes flags: -EINVAL ("option or name mismatch").
- [ ] AC-8: Remount of populated hierarchy: -EBUSY.
- [ ] AC-9: `cat /sys/fs/cgroup/<h>/tasks` returns sorted, deduplicated pid list in caller's pid_namespace.
- [ ] AC-10: Writing pid to `cgroup.procs` from unprivileged user where pid is not the writer's: -EACCES.
- [ ] AC-11: `release_agent` write requires CAP_SYS_ADMIN in init_user_ns.
- [ ] AC-12: Setting `notify_on_release=1` + cgroup emptied: release_agent invoked with cgroup-path argv.
- [ ] AC-13: `/proc/cgroups` lists all enabled legacy controllers with hierarchy_id + nr_cgrps.
- [ ] AC-14: `cgroup_no_v1=memory` boot param: mounting v1 hierarchy with `memory` controller name → "Disabled controller".
- [ ] AC-15: `cgroup_no_v1=named`: mounting `-o name=...` → -ENOENT.
- [ ] AC-16: `cgroup1_rename` on non-directory kernfs node: -ENOTDIR.

## Architecture

```
struct CgroupV1Pidlist {
  key_type: u8,                          // PROCS or TASKS
  key_ns: *PidNamespace,
  list: KVec<Pid>,                       // sorted + dedup
  length: usize,
  links: ListNode,                       // into cgroup.pidlists
  owner: *Cgroup,
  destroy_dwork: DelayedWork,
}

struct CgroupRoot {  // partial view used by v1
  name: [u8; MAX_CGROUP_ROOT_NAMELEN],
  release_agent_path: [u8; PATH_MAX],
  subsys_mask: u32,
  flags: u32,                            // CGRP_ROOT_*
  hierarchy_id: u32,
  nr_cgrps: AtomicI32,
  cgrp: Cgroup,
  ...
}

static cgroup_no_v1_mask: u32 = 0;
static cgroup_no_v1_named: bool = false;
static proc_show_all: bool = false;
static release_agent_path_lock: SpinLock<()>;
static cgroup_pidlist_destroy_wq: *Workqueue;
```

`CgroupV1::parse_param(fc, param) -> i32`:
1. Per-fs_parse → opt + result.
2. If opt == -ENOPARAM:
   - vfs_parse_fs_param_source; if !-ENOPARAM, return that.
   - for_each_subsys(ss, i):
     - if strcmp(param.key, ss.legacy_name) ∨ cgroup1_subsys_absent(ss): continue.
     - if !cgroup_ssid_enabled(i) ∨ Self::ssid_disabled(i): return invalfc("Disabled controller").
     - ctx.subsys_mask |= 1 << i; return 0.
   - Return invalfc("Unknown subsys name").
3. Per-opt dispatch (see REQ-4 switch).

`CgroupV1::check_options(fc) -> i32`:
1. mask = if CONFIG_CPUSETS { !(1 << cpuset_cgrp_id) } else { U32_MAX }.
2. enabled = enabled-and-not-blocked-and-not-absent subsystems.
3. ctx.subsys_mask &= enabled.
4. If empty + !none + !name: all_ss = true.
5. If all_ss: conflict-with-subsys ⟹ invalfc; subsys_mask = enabled.
6. If empty + !name: invalfc.
7. NOPREFIX ∧ (subsys_mask & mask) ⟹ invalfc.
8. subsys_mask ∧ none ⟹ invalfc.

`CgroupV1::root_to_use(fc) -> Result<(), Error>`:
1. Self::check_options.
2. Drain dying-roots for each requested subsys (return 1 to restart on fail).
3. for_each_root(root) excluding dfl:
   - Name match check.
   - Mask match check (mismatch ⟹ -EBUSY only if name matched).
   - Flag mismatch ⟹ warn-ignore.
   - ctx.root = root; return Ok.
4. Else create:
   - !subsys_mask ∧ !none ⟹ invalfc.
   - ctx.ns != init_cgroup_ns ⟹ -EPERM.
   - root = kzalloc; init_cgroup_root(ctx); cgroup_setup_root(root, subsys_mask).
   - On success: cgroup_favor_dynmods(root, flags & FAVOR_DYNMODS).
   - On fail: cgroup_free_root(root).

`CgroupV1::get_tree(fc) -> i32`:
1. ns_capable(ctx.ns.user_ns, CAP_SYS_ADMIN) else -EPERM.
2. cgroup_lock_and_drain_offline(&cgrp_dfl_root.cgrp).
3. Self::root_to_use(fc); if !ret ∧ !tryget_live(refcnt): ret = 1.
4. cgroup_unlock.
5. If !ret: cgroup_do_get_tree(fc).
6. If !ret ∧ percpu_ref_is_dying(refcnt): fc_drop_locked; ret = 1.
7. If ret > 0: msleep(10); restart_syscall.

`CgroupV1::reconfigure(fc) -> i32`:
1. cgroup_lock_and_drain_offline.
2. Self::check_options.
3. Deprecation warn if subsys_mask changed or release_agent set.
4. Compute added_mask / removed_mask.
5. flag-mismatch or name-mismatch ⟹ errorfc + -EINVAL.
6. root.cgrp.self.children non-empty ⟹ -EBUSY.
7. rebind_subsystems(root, added_mask); WARN_ON(rebind_subsystems(&cgrp_dfl_root, removed_mask)).
8. release_agent strscpy under lock.
9. trace_cgroup_remount.

`CgroupV1::procs_write_inner(of, buf, nbytes, off, threadgroup) -> isize`:
1. cgrp = cgroup_kn_lock_live(of.kn, false); else -ENODEV.
2. task = cgroup_procs_write_start(buf, threadgroup, &lock_mode).
3. cred = of.file.f_cred; tcred = get_task_cred(task).
4. !root && cred.euid != tcred.uid && cred.euid != tcred.suid ⟹ -EACCES.
5. put_cred(tcred).
6. cgroup_attach_task(cgrp, task, threadgroup).
7. cgroup_procs_write_finish(task, lock_mode); cgroup_kn_unlock.

`CgroupV1::pidlist_array_load(cgrp, type) -> Result<*Pidlist, i32>`:
1. length = cgroup_task_count(cgrp).
2. array = kvmalloc(pid_t * length); err -ENOMEM.
3. css_task_iter_start; while next:
   - if n == length: break.
   - pid = if PROCS: task_tgid_vnr else task_pid_vnr.
   - if pid > 0: array[n++] = pid.
4. iter_end.
5. sort(array, n, cmppid); n = pidlist_uniq(array, n).
6. l = pidlist_find_create; err -ENOMEM.
7. kvfree(l.list); l.list = array; l.length = n.

`CgroupV1::release_agent_work(work)`:
1. cgrp = container_of(work, Cgroup, release_agent_work).
2. If release_agent_path empty: return.
3. kmalloc pathbuf + agentbuf (PATH_MAX); on alloc fail: free + return.
4. Under release_agent_path_lock: strscpy(agentbuf, root.release_agent_path).
5. If agentbuf empty: free + return.
6. cgroup_path_ns(cgrp, pathbuf, PATH_MAX, &init_cgroup_ns); ret < 0 ⟹ free + return.
7. argv = [agentbuf, pathbuf, NULL]; envp = ["HOME=/", "PATH=/sbin:/bin:/usr/sbin:/usr/bin", NULL].
8. call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC).
9. kfree(agentbuf); kfree(pathbuf).

`CgroupV1::check_for_release(cgrp)`:
1. If notify_on_release(cgrp) ∧ !cgroup_is_populated(cgrp) ∧ !css_has_online_children(&cgrp.self) ∧ !cgroup_is_dead(cgrp):
   - schedule_work(&cgrp.release_agent_work).

`CgroupV1::attach_task_all(from, tsk) -> i32`:
1. cgroup_lock; cgroup_attach_lock(GLOBAL).
2. for_each_root(root):
   - from_cgrp = task_cgroup_from_root(from, root) under css_set_lock.
   - cgroup_attach_task(from_cgrp, tsk, false); on err: break.
3. cgroup_attach_unlock(GLOBAL); cgroup_unlock.

`CgroupV1::task_get_cgroup1(tsk, hierarchy_id) -> *Cgroup`:
1. rcu_read_lock.
2. for_each_root(root):
   - skip dfl.
   - if hierarchy_id mismatch: continue.
   - css_set_lock irqsave; cgrp = task_cgroup_from_root(tsk, root).
   - if !cgrp ∨ !cgroup_tryget(cgrp): cgrp = ERR_PTR(-ENOENT).
   - break.
3. rcu_read_unlock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pidlist_array_sorted_dedup` | INVARIANT | per-pidlist_array_load: post-condition list strictly increasing. |
| `release_agent_path_under_lock` | INVARIANT | per-read/write of root.release_agent_path: release_agent_path_lock held. |
| `mount_subsys_mask_subset_of_enabled` | INVARIANT | per-root_to_use: subsys_mask ⊆ enabled-mask. |
| `name_charset_validated` | INVARIANT | per-parse_param Opt_name: every char ∈ [\w.-]. |
| `noprefix_only_cpuset` | INVARIANT | per-check_options: NOPREFIX ⟹ subsys_mask ⊆ {cpuset}. |
| `reconfigure_rejects_populated_root` | INVARIANT | per-reconfigure: !empty(children) ⟹ -EBUSY. |
| `procs_write_capability_check` | INVARIANT | per-procs_write_inner: euid match or root before attach. |
| `release_agent_write_requires_caps` | INVARIANT | per-cgroup_release_agent_write: init_user_ns + CAP_SYS_ADMIN. |
| `rename_no_newline` | INVARIANT | per-cgroup1_rename: '\n' ⟹ -EINVAL. |
| `rename_same_parent` | INVARIANT | per-cgroup1_rename: kn.__parent == new_parent. |

### Layer 2: TLA+

`kernel/cgroup/cgroup-v1.tla`:
- Per-mount: parse_param → check_options → root_to_use → get_tree → setup_root.
- Per-remount: reconfigure → rebind_subsystems.
- Per-pidlist: load → iter → delayed-destroy.
- Per-release-agent: check_for_release → schedule_work → usermodehelper.
- Properties:
  - `safety_named_hierarchy_unique` — per-mount: two roots never share `name` with different subsys_mask without -EBUSY.
  - `safety_dfl_root_excluded` — per-v1-ops: cgrp_dfl_root never enumerated.
  - `safety_release_agent_only_init_userns` — per-write/parse: release_agent always in init_user_ns.
  - `safety_remount_populated_blocked` — per-reconfigure: children non-empty ⟹ no rebind.
  - `liveness_release_agent_invoked_on_empty` — per-empty-cgroup: usermodehelper eventually scheduled.
  - `liveness_pidlist_destroyed_after_delay` — per-pidlist: destroyed within `CGROUP_PIDLIST_DESTROY_DELAY` after last access.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `CgroupV1::parse_param` post: ctx.subsys_mask only contains enabled+allowed ssids | `CgroupV1::parse_param` |
| `CgroupV1::check_options` post: invariants of REQ-5 hold | `CgroupV1::check_options` |
| `CgroupV1::root_to_use` post: ctx.root != cgrp_dfl_root ∧ (existing match OR new root) | `CgroupV1::root_to_use` |
| `CgroupV1::reconfigure` post: subsys_mask updated; flags + name unchanged | `CgroupV1::reconfigure` |
| `CgroupV1::pidlist_array_load` post: sorted + unique + pid > 0 only | `CgroupV1::pidlist_array_load` |
| `CgroupV1::release_agent_work` post: usermodehelper invoked exactly once per release event | `CgroupV1::release_agent_work` |
| `CgroupV1::attach_task_all` post: for every v1 root, tsk in same cgroup as from | `CgroupV1::attach_task_all` |
| `CgroupV1::task_get_cgroup1` post: returned cgrp.root.hierarchy_id == hierarchy_id | `CgroupV1::task_get_cgroup1` |

### Layer 4: Verus/Creusot functional

`Per-mount(opts) → root_lookup_or_create → bind_subsystems → kernfs_tree`, `per-remount(opts) → rebind`, `per-task_count → pidlist_load → seq_show`, `per-empty(cgrp) → schedule_release_agent → call_usermodehelper(cgrp_path)` semantic equivalence: per-Documentation/admin-guide/cgroup-v1/cgroups.rst + per-Documentation/admin-guide/cgroup-v1/cpusets.rst.

## Hardening

(Inherits row-1 features from `kernel/cgroup/00-overview.md` § Hardening.)

cgroup-v1 reinforcement:

- **Per-mount init_cgroup_ns required for fresh root** — defense against per-userns-spoofed-hierarchy creation.
- **Per-name charset `[\w.-]+` + max length** — defense against per-/proc/<pid>/cgroup parser corruption.
- **Per-release_agent CAP_SYS_ADMIN + init_user_ns** — defense against per-unprivileged-usermodehelper-injection.
- **Per-release_agent path lock + strscpy** — defense against per-TOCTOU during agent invocation.
- **Per-procs/tasks write euid/uid/suid check via of->file->f_cred** — defense against per-inherited-fd capability escalation.
- **Per-NOPREFIX restricted to cpuset-only mounts** — defense against per-mixed-controller-file-collision.
- **Per-remount rejects flag/name change + populated hierarchies** — defense against per-live-hierarchy semantic drift.
- **Per-pidlist per-pid-namespace keyed** — defense against per-cross-ns pid leak.
- **Per-pidlist delayed destroy + flush on cgroup destroy** — defense against per-iterator UAF.
- **Per-rename reject '\n' + same-parent only** — defense against per-/proc parser corruption + cross-hierarchy reparent.
- **Per-cgroup_no_v1= boot gate (incl. `named`)** — defense against per-legacy-surface attack on hardened systems.
- **Per-call_usermodehelper UMH_WAIT_EXEC bounded** — defense against per-release-agent-hang DoS.
- **Per-attach_task_all under CGRP_ATTACH_LOCK_GLOBAL** — defense against per-partial-migration race.

## Open Questions

- Should Rookery deprecate v1 mount entirely behind a Kconfig (forcing v2-only) given upstream's repeated calls to retire v1? Default: keep behind `CONFIG_CGROUP_V1` gated by `cgroup_no_v1=all`.
- How are per-pid-namespace pidlists garbage-collected when a pid_ns is destroyed independently of the cgroup? Document the get_pid_ns ref-counting tie-in.

## Out of Scope

- cgroup v2 unified hierarchy (covered in `cgroup-core.md` Tier-3)
- Individual v1 controllers (cpuset, memory, blkio, devices, freezer, perf_event, hugetlb, net_cls, net_prio, pids) — each covered separately (see `cpuset.md`, `pids.md`, `freezer.md`, ...)
- `rebind_subsystems` itself (covered in `cgroup-core.md` Tier-3)
- `cgroup_attach_task` migration machinery (covered in `cgroup-core.md` Tier-3)
- BPF cgroup attachment (covered separately if expanded)
- Implementation code
