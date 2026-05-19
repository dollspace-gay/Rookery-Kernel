# Tier-3: kernel/pid_namespace.c — PID namespace lifecycle

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/pid_namespace.c (~473 lines)
  - include/linux/pid_namespace.h
  - include/linux/pid.h
  - include/linux/proc_ns.h
  - kernel/pid.c (sibling — pid allocation; not covered here)
-->

## Summary

A **PID namespace** is the isolation unit that gives a process tree its own pid 1 (init), its own translation table (`struct idr`) of pid numbers, and its own `/proc` view. Nested up to `MAX_PID_NS_LEVEL` (32) levels deep. Per-`struct pid_namespace`: idr (per-numeric allocator), parent (per-ancestor ns), level (per-depth), user_ns (per-owning user-ns), child_reaper (per-init task), pid_allocated (per-active-count + PIDNS_ADDING/_HASH_ADDING flags), pid_max (per-ns ceiling), reboot (per-PID 1 reboot intent), ucounts (per-RLIMIT_NPROC slot). Per-`pid_ns_for_children`: per-`task.nsproxy` slot that holds the PID namespace into which the **next** `fork()` allocates pids (decoupled from `task_active_pid_ns(current)` so `setns()` + `fork()` enters a new ns without crossing yourself). Per-`copy_pid_ns(flags, user_ns, old)`: per-fork dispatch — `CLONE_NEWPID` triggers `create_pid_namespace(user_ns, old)`. Per-`zap_pid_ns_processes(pid_ns)`: per-init-exit cleanup — disable allocation, SIGKILL all remaining tasks, reap zombies, account exit. Per-`/proc/<pid>/ns/pid` symlink: per-task observable handle via `pidns_operations.{get,put,install,owner,get_parent}` (and the parallel `pidns_for_children_operations` for `pid_for_children`). Critical for: container PID isolation, init-process protection, reliable container shutdown, CRIU checkpoint/restore.

This Tier-3 covers `kernel/pid_namespace.c` (~473 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pid_namespace` | per-ns descriptor | `PidNamespace` |
| `init_pid_ns` | per-boot init ns | `INIT_PID_NS` (static) |
| `MAX_PID_NS_LEVEL` (32) | per-depth ceiling | `MAX_PID_NS_LEVEL` |
| `pid_ns_cachep` | per-slab cache for pid_namespace | `PidNs::SLAB` |
| `pid_cache[level]` | per-level slab cache for `struct pid` | `PidNs::PID_CACHE` |
| `create_pid_cachep(level)` | per-level cache lazy-init | `PidNs::create_pid_cachep` |
| `inc_pid_namespaces(user_ns)` / `dec_pid_namespaces(uc)` | per-ucounts RLIMIT_NPROC accounting | `PidNs::inc` / `PidNs::dec` |
| `create_pid_namespace(user_ns, parent)` | per-construct | `PidNs::create` |
| `copy_pid_ns(flags, user_ns, old)` | per-fork CLONE_NEWPID dispatch | `PidNs::copy` |
| `delayed_free_pidns(rcu_head)` | per-RCU-free callback | `PidNs::delayed_free` |
| `destroy_pid_namespace(ns)` | per-tree-remove + idr_destroy + RCU-free | `PidNs::destroy` |
| `destroy_pid_namespace_work(work)` | per-workqueue cascading destroy | `PidNs::destroy_work` |
| `put_pid_ns(ns)` | per-ref-drop ⟹ schedule_work(destroy) | `PidNs::put` |
| `get_pid_ns(ns)` | per-ref-grab | `PidNs::get` |
| `zap_pid_ns_processes(pid_ns)` | per-init-exit cleanup | `PidNs::zap_processes` |
| `disable_pid_allocation(pid_ns)` | per-PIDNS_ADDING clear | `PidNs::disable_allocation` |
| `reboot_pid_ns(pid_ns, cmd)` | per-PID 1 reboot intent | `PidNs::reboot` |
| `pidns_get(task)` | per-`/proc/<pid>/ns/pid` getter | `PidNs::ns_get` |
| `pidns_for_children_get(task)` | per-`/proc/<pid>/ns/pid_for_children` getter | `PidNs::ns_for_children_get` |
| `pidns_put(ns_common)` | per-`/proc/<pid>/ns/pid` putter | `PidNs::ns_put` |
| `pidns_install(nsset, ns_common)` | per-`setns(2)` install | `PidNs::ns_install` |
| `pidns_get_parent(ns_common)` | per-`ioctl(NS_GET_PARENT)` | `PidNs::ns_get_parent` |
| `pidns_owner(ns_common)` | per-`ioctl(NS_GET_USERNS)` | `PidNs::ns_owner` |
| `pidns_is_ancestor(child, ancestor)` | per-install ancestor check | `PidNs::is_ancestor` |
| `pidns_operations` | per-`/proc/.../ns/pid` ops vector | `PidNs::PROC_OPS` |
| `pidns_for_children_operations` | per-`/proc/.../ns/pid_for_children` ops vector | `PidNs::PROC_OPS_FOR_CHILDREN` |
| `pid_ns_ctl_handler` (CONFIG_CHECKPOINT_RESTORE) | per-`ns_last_pid` sysctl | `PidNs::ctl_handler` |
| `pid_ns_ctl_table` | per-sysctl declarations | `PidNs::CTL_TABLE` |
| `pid_namespaces_init` (initcall) | per-boot setup | `PidNs::init` |
| `PIDNS_ADDING` | per-`pid_allocated` flag: allocations permitted | `PIDNS_ADDING` |

## Compatibility contract

REQ-1: `struct pid_namespace`:
- idr: `struct idr` — per-pid-number allocator (1..pid_max - 1).
- pid_allocated: u32 — `PIDNS_ADDING` bit + low bits = count of allocated pids; clearing PIDNS_ADDING blocks new allocations.
- level: u32 — depth from init_pid_ns (init = 0, child = 1, ...).
- parent: `*PidNamespace` — per-tree edge (NULL only for init_pid_ns).
- user_ns: `*UserNamespace` — per-owner; capability checks scope to this user-ns.
- ucounts: `*Ucounts` — RLIMIT_NPROC slot in parent user-ns.
- pid_cachep: `*KmemCache` — slab for `struct pid` at this level (`pid_cache[level - 1]`).
- pid_max: i32 — ceiling for idr allocation; default `PID_MAX_LIMIT` (4 * 1024 * 1024).
- reboot: i32 — set by `reboot_pid_ns()`; consumed by init's `do_exit()` as `group_exit_code`.
- child_reaper: `*TaskStruct` — per-pid-1; receives reparented orphans and SIGKILL at zap-time.
- work: `struct work_struct` — destroy_pid_namespace_work (cascading via parent).
- rcu: `struct rcu_head` — delayed_free_pidns.
- memfd_noexec_scope (CONFIG_SYSCTL+CONFIG_MEMFD_CREATE): per-ns memfd-noexec inheritance.
- ns: `struct ns_common` — per-/proc/.../ns/pid backing (inum, ops, ref).

REQ-2: `init_pid_ns` (`static struct pid_namespace`):
- level = 0; parent = NULL; user_ns = &init_user_ns; pid_max = PID_MAX_LIMIT.
- Globally singular; never freed.
- Always referenced by container_of'd tasks at the top of the per-task `pid` chain (`numbers[0]`).

REQ-3: `MAX_PID_NS_LEVEL` (32):
- `create_pid_namespace()`: `level = parent.level + 1`; if `level > MAX_PID_NS_LEVEL` ⟹ `-ENOSPC`.
- Bounds `struct pid.numbers[]` array; per-numeric translation cost is O(level).

REQ-4: `create_pid_cachep(level)`:
- Lazy per-level slab cache for `struct pid` (`pid_<level+1>`).
- size = `struct_size_t(struct pid, numbers, level + 1)` — flex-array per nested depth.
- mutex_lock(&pid_caches_mutex) for first-time creation; `kmem_cache_create(..., SLAB_HWCACHE_ALIGN | SLAB_ACCOUNT, NULL)`.
- READ_ONCE for fast-path lookup after creation.
- `pid_cache[0]` = level 1 (init's children); `init_pid_ns.pid_cachep` is set up separately at boot.

REQ-5: `create_pid_namespace(user_ns, parent_pid_ns)`:
- `in_userns(parent.user_ns, user_ns)` required (user_ns must be parent.user_ns or descendant) ⟹ else `-EINVAL`.
- level > MAX_PID_NS_LEVEL ⟹ `-ENOSPC`.
- `inc_pid_namespaces(user_ns)`: `inc_ucount(user_ns, current_euid(), UCOUNT_PID_NAMESPACES)`; NULL ⟹ `-ENOSPC` (rlimit exceeded).
- `kmem_cache_zalloc(pid_ns_cachep, GFP_KERNEL)`; NULL ⟹ `-ENOMEM`.
- idr_init(&ns.idr).
- ns.pid_cachep = create_pid_cachep(level); NULL ⟹ free-and-fail.
- `ns_common_init(ns)` allocates an inum (proc-ns identifier).
- ns.pid_max = `PID_MAX_LIMIT`.
- `register_pidns_sysctls(ns)` (per-ns `/proc/sys/kernel/{ns_last_pid,pid_max,...}`).
- ns.level = level; ns.parent = `get_pid_ns(parent_pid_ns)`; ns.user_ns = `get_user_ns(user_ns)`; ns.ucounts = ucounts.
- ns.pid_allocated = `PIDNS_ADDING` (allocations allowed; count = 0).
- INIT_WORK(&ns.work, destroy_pid_namespace_work).
- (CONFIG_SYSCTL+CONFIG_MEMFD_CREATE): ns.memfd_noexec_scope = parent_pid_ns.memfd_noexec_scope.
- `ns_tree_add(ns)` (publish in the global nstree, so /proc enumeration can find it).
- return ns.
- Failure path unwinds in reverse: ns_common_free → idr_destroy → kmem_cache_free → dec_pid_namespaces.

REQ-6: `copy_pid_ns(flags, user_ns, old_ns)` — called from `copy_namespaces()` at fork:
- `!(flags & CLONE_NEWPID)` ⟹ `get_pid_ns(old_ns)` (share).
- `task_active_pid_ns(current) != old_ns` ⟹ `-EINVAL` (can only create child of own active ns).
- else `create_pid_namespace(user_ns, old_ns)`.

REQ-7: `put_pid_ns(ns)`:
- ns == NULL ⟹ no-op.
- `ns_ref_put(ns)` (atomic_dec_and_test on ns.ns.count): false ⟹ no-op.
- true ⟹ `schedule_work(&ns.work)` — destroy in workqueue context (must not free synchronously: holds locks / RCU dependencies; cascades up tree).

REQ-8: `destroy_pid_namespace(ns)`:
- `ns_tree_remove(ns)`.
- `unregister_pidns_sysctls(ns)`.
- `ns_common_free(ns)` (release inum).
- `idr_destroy(&ns.idr)` — at this point all pids have been freed.
- `call_rcu(&ns.rcu, delayed_free_pidns)` — RCU-delayed final free (readers of `pid.numbers[].ns` may still be in flight).

REQ-9: `delayed_free_pidns(rcu_head)`:
- `dec_pid_namespaces(ns.ucounts)` (free RLIMIT NPROC slot).
- `put_user_ns(ns.user_ns)`.
- `kmem_cache_free(pid_ns_cachep, ns)`.

REQ-10: `destroy_pid_namespace_work(work)`:
- Loop: parent = ns.parent; destroy_pid_namespace(ns); ns = parent.
- Continue while `ns != &init_pid_ns ∧ ns_ref_put(ns)` (cascading destroy: dropping last child reference may now drop parent's last reference).
- Bounded by `MAX_PID_NS_LEVEL` iterations.

REQ-11: `zap_pid_ns_processes(pid_ns)` — called by `do_exit()` when the cgroup-init thread group terminates:
- `disable_pid_allocation(pid_ns)`: clear PIDNS_ADDING bit (no new fork/clone into this ns).
- `spin_lock_irq(&me.sighand.siglock); me.sighand.action[SIGCHLD - 1].sa.sa_handler = SIG_IGN; spin_unlock_irq` — auto-reap children to speed shutdown.
- init_pids = `thread_group_leader(me) ? 1 : 2` (count of pids we hold).
- rcu_read_lock + read_lock(&tasklist_lock).
- `idr_for_each_entry_continue(&pid_ns.idr, pid, 2)`: SIGKILL every remaining task that doesn't already have a fatal signal pending: `group_send_sig_info(SIGKILL, SEND_SIG_PRIV, task, PIDTYPE_MAX)`.
- read_unlock + rcu_read_unlock.
- Loop: clear TIF_SIGPENDING/TIF_NOTIFY_SIGNAL; `kernel_wait4(-1, NULL, __WALL, NULL)` until `-ECHILD` (reap all zombies, including ptraced from parent ns).
- Loop: TASK_INTERRUPTIBLE; break when `pid_ns.pid_allocated == init_pids` (only init + thread leader pid remain).
- if pid_ns.reboot != 0: `current.signal.group_exit_code = pid_ns.reboot` (propagate reboot intent to parent's wait status).
- `acct_exit_ns(pid_ns)` (BSD process accounting).

REQ-12: `reboot_pid_ns(pid_ns, cmd)` — called from `sys_reboot` when caller is not in init_pid_ns:
- pid_ns == &init_pid_ns ⟹ 0 (allow normal reboot path).
- switch cmd:
  - `LINUX_REBOOT_CMD_RESTART` / `_RESTART2` ⟹ pid_ns.reboot = SIGHUP.
  - `LINUX_REBOOT_CMD_POWER_OFF` / `_HALT` ⟹ pid_ns.reboot = SIGINT.
  - default ⟹ `-EINVAL`.
- read_lock(&tasklist_lock); `send_sig(SIGKILL, pid_ns.child_reaper, 1)`; read_unlock — kill init; zap-path then propagates the reboot intent up.
- `do_exit(0)` — caller (typically init) exits; never returns.

REQ-13: `pidns_get(task)` — `/proc/<pid>/ns/pid` open getter:
- rcu_read_lock; `ns = task_active_pid_ns(task)`; if ns: get_pid_ns(ns); rcu_read_unlock.
- return `&ns.ns` (`struct ns_common`), or NULL if dead.

REQ-14: `pidns_for_children_get(task)` — `/proc/<pid>/ns/pid_for_children` open getter:
- task_lock(task); if task.nsproxy: ns = task.nsproxy.pid_ns_for_children; get_pid_ns(ns); task_unlock.
- return `&ns.ns` or NULL.

REQ-15: `pidns_install(nsset, ns_common)` — `setns(2)` install:
- active = task_active_pid_ns(current); new = to_pid_ns(ns_common).
- `!ns_capable(new.user_ns, CAP_SYS_ADMIN) ∨ !ns_capable(nsset.cred.user_ns, CAP_SYS_ADMIN)` ⟹ `-EPERM`.
- `!pidns_is_ancestor(new, active)` ⟹ `-EINVAL` — caller may only enter a descendant of (or equal to) its own active pid-ns; **forbids ascending** (per-property: a process cannot escape its pid-ns).
- `put_pid_ns(nsproxy.pid_ns_for_children); nsproxy.pid_ns_for_children = get_pid_ns(new)` — installs only into `pid_for_children`; the caller's own active pid-ns is unchanged; only the **next fork()** lands in `new`.

REQ-16: `pidns_is_ancestor(child, ancestor)`:
- `child.level < ancestor.level` ⟹ false.
- Walk child.parent until level == ancestor.level; return ns == ancestor.

REQ-17: `pidns_get_parent(ns_common)` — `ioctl(NS_GET_PARENT)`:
- Walk to.parent; find first ancestor that is == active (current's pid-ns).
- If none, `-EPERM` (parent is above caller's visibility).
- Return parent_ref.

REQ-18: `pidns_owner(ns_common)` — `ioctl(NS_GET_USERNS)`: return `to_pid_ns(ns).user_ns`.

REQ-19: `pid_ns_ctl_handler` (CONFIG_CHECKPOINT_RESTORE) — `/proc/sys/kernel/ns_last_pid`:
- write requires `checkpoint_restore_ns_capable(pid_ns.user_ns)`.
- read: returns `idr_get_cursor(&pid_ns.idr) - 1`.
- write: set cursor to `next + 1` so next pid allocation produces `next + 1` (CRIU restore — recreate exact pid numbers).
- proc_dointvec_minmax with extra2 = &pid_ns.pid_max.

REQ-20: `pid_namespaces_init` (initcall):
- pid_ns_cachep = `KMEM_CACHE(pid_namespace, SLAB_PANIC | SLAB_ACCOUNT)`.
- (CONFIG_CHECKPOINT_RESTORE) `register_sysctl_init("kernel", pid_ns_ctl_table)`.
- `register_pid_ns_sysctl_table_vm()`.
- `ns_tree_add(&init_pid_ns)`.

## Acceptance Criteria

- [ ] AC-1: `clone(CLONE_NEWPID)` returns child with `getpid() == 1` inside, mapped to a different pid in parent.
- [ ] AC-2: `clone(CLONE_NEWPID)` at depth > 32 returns `-ENOSPC`.
- [ ] AC-3: `clone(CLONE_NEWPID)` requires `CAP_SYS_ADMIN` over the active user-ns (via per-`copy_namespaces` enforcement).
- [ ] AC-4: `setns(fd_of_descendant_pid_ns, CLONE_NEWPID)` then `fork()` ⟹ child in the descendant.
- [ ] AC-5: `setns(fd_of_ancestor_pid_ns, CLONE_NEWPID)` ⟹ `-EINVAL` (cannot ascend).
- [ ] AC-6: `setns` without `CAP_SYS_ADMIN` on the target's user-ns ⟹ `-EPERM`.
- [ ] AC-7: pid 1 in a child ns exits ⟹ `zap_pid_ns_processes` SIGKILLs all other pids in that ns; parent's `waitpid` returns the ns's `group_exit_code`.
- [ ] AC-8: After pid 1 exit, fork into that ns ⟹ `-ENOMEM` (PIDNS_ADDING cleared, no new pids allocatable).
- [ ] AC-9: `reboot(RB_RESTART)` from non-init pid-ns ⟹ kills its init with SIGKILL, sets `group_exit_code = SIGHUP`, parent observes via WIFSIGNALED + WTERMSIG == SIGHUP.
- [ ] AC-10: `reboot(RB_POWER_OFF)` from non-init pid-ns ⟹ `group_exit_code = SIGINT`.
- [ ] AC-11: `/proc/<pid>/ns/pid` symlink stat-inum matches `pid_ns.ns.inum`; `readlink` returns `pid:[inum]` format.
- [ ] AC-12: `/proc/<pid>/ns/pid_for_children` reflects `task.nsproxy.pid_ns_for_children` (may differ from `ns/pid` after `setns`).
- [ ] AC-13: `ioctl(NS_GET_PARENT)` returns parent's ns_fd; from init_pid_ns ⟹ `-EPERM`.
- [ ] AC-14: `ioctl(NS_GET_USERNS)` returns owning user-ns ns_fd.
- [ ] AC-15: `RLIMIT_NPROC` for `UCOUNT_PID_NAMESPACES` exceeded ⟹ `clone(CLONE_NEWPID)` returns `-ENOSPC`.
- [ ] AC-16: Writes to `/proc/sys/kernel/ns_last_pid` without `CAP_CHECKPOINT_RESTORE` over pid-ns's user-ns ⟹ `-EPERM`; reads return last-allocated pid.

## Architecture

```
pub struct PidNamespace {
  pub ns: NsCommon,
  pub idr: Idr,                  /* per-pid-number allocator */
  pub pid_allocated: AtomicU32,  /* PIDNS_ADDING | count */
  pub pid_max: AtomicI32,
  pub level: u32,
  pub parent: *const PidNamespace,
  pub user_ns: *const UserNamespace,
  pub ucounts: *const Ucounts,
  pub pid_cachep: *const KmemCache,
  pub reboot: AtomicI32,
  pub child_reaper: *const TaskStruct, /* per-pid-1 */
  pub work: WorkStruct,                /* destroy_pid_namespace_work */
  pub rcu: RcuHead,                    /* delayed_free_pidns */
  #[cfg(all(sysctl, memfd_create))]
  pub memfd_noexec_scope: i32,
}

pub const MAX_PID_NS_LEVEL: u32 = 32;
pub const PIDNS_ADDING: u32 = 1 << 31;

pub static INIT_PID_NS: PidNamespace = /* level=0, parent=NULL, user_ns=&INIT_USER_NS, ... */;
```

`PidNs::create(user_ns, parent) -> Result<*PidNamespace, Errno>`:
1. let level = parent.level + 1.
2. if !in_userns(parent.user_ns, user_ns) { return Err(EINVAL) }.
3. if level > MAX_PID_NS_LEVEL { return Err(ENOSPC) }.
4. let ucounts = inc_pid_namespaces(user_ns).ok_or(ENOSPC)?
5. let ns = kmem_cache_zalloc(pid_ns_cachep, GFP_KERNEL).ok_or(ENOMEM)?
6. idr_init(&mut ns.idr).
7. ns.pid_cachep = PidNs::create_pid_cachep(level).ok_or(ENOMEM)?
8. ns_common_init(&mut ns.ns)?
9. ns.pid_max = PID_MAX_LIMIT.
10. register_pidns_sysctls(&mut ns)?
11. ns.level = level; ns.parent = get_pid_ns(parent); ns.user_ns = get_user_ns(user_ns); ns.ucounts = ucounts.
12. ns.pid_allocated = PIDNS_ADDING.
13. INIT_WORK(&mut ns.work, PidNs::destroy_work).
14. #[cfg(memfd)] ns.memfd_noexec_scope = pidns_memfd_noexec_scope(parent).
15. ns_tree_add(&ns).
16. Ok(ns).
/* Failure unwinds inverse, ending with dec_pid_namespaces(ucounts). */

`PidNs::copy(flags, user_ns, old_ns) -> Result<*PidNamespace, Errno>`:
1. if (flags & CLONE_NEWPID) == 0 { return Ok(get_pid_ns(old_ns)) }.
2. if task_active_pid_ns(current()) != old_ns { return Err(EINVAL) }.
3. PidNs::create(user_ns, old_ns).

`PidNs::put(ns)`:
1. if ns.is_null() { return }.
2. if ns_ref_put(&ns.ns) { schedule_work(&ns.work) }.

`PidNs::destroy_work(work)`:
1. let mut ns = container_of(work, PidNamespace, work).
2. loop {
   - let parent = ns.parent.
   - PidNs::destroy(ns).
   - ns = parent.
   - if ns == &INIT_PID_NS || !ns_ref_put(&ns.ns) { break }.
   }.

`PidNs::destroy(ns)`:
1. ns_tree_remove(ns).
2. unregister_pidns_sysctls(ns).
3. ns_common_free(&ns.ns).
4. idr_destroy(&mut ns.idr).
5. call_rcu(&ns.rcu, PidNs::delayed_free).

`PidNs::delayed_free(rcu)`:
1. let ns = container_of(rcu, PidNamespace, rcu).
2. dec_pid_namespaces(ns.ucounts).
3. put_user_ns(ns.user_ns).
4. kmem_cache_free(pid_ns_cachep, ns).

`PidNs::zap_processes(pid_ns)`:
1. disable_pid_allocation(pid_ns)  /* clear PIDNS_ADDING */.
2. spin_lock_irq(&current.sighand.siglock).
3. current.sighand.action[SIGCHLD - 1].sa.sa_handler = SIG_IGN.
4. spin_unlock_irq.
5. let init_pids = if thread_group_leader(current) { 1 } else { 2 }.
6. rcu_read_lock; read_lock(&tasklist_lock).
7. idr_for_each_entry_continue(&pid_ns.idr, pid, 2) {
   - let task = pid_task(pid, PIDTYPE_PID).
   - if task && !__fatal_signal_pending(task) {
     group_send_sig_info(SIGKILL, SEND_SIG_PRIV, task, PIDTYPE_MAX);
     }
   }.
8. read_unlock; rcu_read_unlock.
9. loop {
   - clear_thread_flag(TIF_SIGPENDING); clear_thread_flag(TIF_NOTIFY_SIGNAL).
   - rc = kernel_wait4(-1, NULL, __WALL, NULL).
   - if rc == -ECHILD { break }.
   }.
10. loop {
    - set_current_state(TASK_INTERRUPTIBLE).
    - if pid_ns.pid_allocated == init_pids { break }.
    - schedule().
    }.
11. __set_current_state(TASK_RUNNING).
12. if pid_ns.reboot != 0 { current.signal.group_exit_code = pid_ns.reboot }.
13. acct_exit_ns(pid_ns).

`PidNs::reboot(pid_ns, cmd) -> i32`:
1. if pid_ns == &INIT_PID_NS { return 0 }.
2. pid_ns.reboot = match cmd {
   LINUX_REBOOT_CMD_RESTART | LINUX_REBOOT_CMD_RESTART2 => SIGHUP,
   LINUX_REBOOT_CMD_POWER_OFF | LINUX_REBOOT_CMD_HALT  => SIGINT,
   _ => return -EINVAL,
   }.
3. read_lock(&tasklist_lock); send_sig(SIGKILL, pid_ns.child_reaper, 1); read_unlock.
4. do_exit(0); /* unreachable */.

`PidNs::ns_install(nsset, ns_common) -> Result<(), Errno>`:
1. let active = task_active_pid_ns(current()).
2. let new = to_pid_ns(ns_common).
3. if !ns_capable(new.user_ns, CAP_SYS_ADMIN) || !ns_capable(nsset.cred.user_ns, CAP_SYS_ADMIN) { return Err(EPERM) }.
4. if !PidNs::is_ancestor(new, active) { return Err(EINVAL) }.
5. put_pid_ns(nsset.nsproxy.pid_ns_for_children).
6. nsset.nsproxy.pid_ns_for_children = get_pid_ns(new).
7. Ok(()).

`PidNs::is_ancestor(child, ancestor) -> bool`:
1. if child.level < ancestor.level { return false }.
2. let mut ns = child; while ns.level > ancestor.level { ns = ns.parent }.
3. ns == ancestor.

`PidNs::ns_get_parent(ns_common) -> Result<*NsCommon, Errno>`:
1. let active = task_active_pid_ns(current()).
2. let mut p = to_pid_ns(ns_common).parent.
3. loop {
   - if p.is_null() { return Err(EPERM) }.
   - if p == active { break }.
   - p = p.parent.
   }.
4. Ok(&get_pid_ns(p).ns).

`PidNs::PROC_OPS: ProcNsOperations` = {
  name: "pid", get: pidns_get, put: pidns_put, install: pidns_install, owner: pidns_owner, get_parent: pidns_get_parent,
}.

`PidNs::PROC_OPS_FOR_CHILDREN: ProcNsOperations` = {
  name: "pid_for_children", real_ns_name: "pid",
  get: pidns_for_children_get, put: pidns_put, install: pidns_install, owner: pidns_owner, get_parent: pidns_get_parent,
}.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `level_bounded` | INVARIANT | per-create: ns.level ∈ [1, MAX_PID_NS_LEVEL]. |
| `parent_ref_held_while_alive` | INVARIANT | per-create..destroy: get_pid_ns(parent) balanced by destroy chain. |
| `user_ns_ref_held_while_alive` | INVARIANT | per-create..delayed_free: get_user_ns / put_user_ns balanced. |
| `ucounts_balanced` | INVARIANT | per-inc_pid_namespaces..dec_pid_namespaces balanced (delayed_free path). |
| `install_no_ascend` | INVARIANT | per-install: PidNs::is_ancestor(new, active) holds before pid_ns_for_children swap. |
| `install_requires_caps` | INVARIANT | per-install: ns_capable(new.user_ns, CAP_SYS_ADMIN) ∧ ns_capable(cred.user_ns, CAP_SYS_ADMIN). |
| `zap_disables_allocation_first` | INVARIANT | per-zap: disable_pid_allocation(pid_ns) before idr scan. |
| `zap_terminates_on_init_pids` | INVARIANT | per-zap: terminates when pid_allocated == init_pids. |
| `reboot_only_non_init` | INVARIANT | per-reboot: pid_ns != &INIT_PID_NS for non-zero path. |
| `destroy_cascade_bounded` | INVARIANT | per-destroy_work: loop iterations ≤ MAX_PID_NS_LEVEL. |

### Layer 2: TLA+

`kernel/pid-namespace.tla`:
- Per-state: forest of pid_namespaces, per-ns idr, per-task active pid-ns and pid_ns_for_children, per-fork pid_allocated.
- Per-action: Create / Fork / Setns / InitExit / Reboot / Destroy.
- Properties:
  - `safety_no_escape_via_setns` — per-Setns: ∀ tasks. active_pid_ns is descendant of original active.
  - `safety_max_depth` — per-Create: level ≤ MAX_PID_NS_LEVEL.
  - `safety_init_pid_ns_immortal` — per-state: INIT_PID_NS.refcount ≥ 1.
  - `safety_no_alloc_after_init_exit` — per-zap..destroy: PIDNS_ADDING cleared ⟹ no new fork into ns.
  - `safety_reboot_signal_propagates` — per-Reboot: group_exit_code ∈ {SIGHUP, SIGINT} observed by parent waitpid.
  - `liveness_zap_completes` — per-zap: under fairness, terminates (no live tasks remain).
  - `liveness_destroy_cascade_terminates` — per-destroy_work: bounded by depth.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `PidNs::create` post: ns.level == parent.level + 1 ∧ ns.parent == parent ∧ ns.pid_allocated == PIDNS_ADDING ∧ ns_tree contains ns | `PidNs::create` |
| `PidNs::copy` post: (flags & CLONE_NEWPID == 0) ⟹ ret == old_ns_with_ref+1; else ret.level == old_ns.level + 1 | `PidNs::copy` |
| `PidNs::put` post: ret-from-ns_ref_put ⟹ destroy work scheduled | `PidNs::put` |
| `PidNs::destroy_work` post: all ns from start..init removed from tree, ucounts dec'd, slab freed (after RCU) | `PidNs::destroy_work` |
| `PidNs::zap_processes` post: pid_ns.pid_allocated == init_pids ∧ all non-init tasks reaped | `PidNs::zap_processes` |
| `PidNs::ns_install` post: !err ⟹ nsproxy.pid_ns_for_children == new ∧ PidNs::is_ancestor(new, active) | `PidNs::ns_install` |
| `PidNs::is_ancestor` post: terminates in ≤ MAX_PID_NS_LEVEL steps ∧ correctly returns child ∈ ancestor.descendants | `PidNs::is_ancestor` |
| `PidNs::reboot` post: pid_ns != INIT ⟹ pid_ns.reboot ∈ {SIGHUP, SIGINT} ∧ child_reaper got SIGKILL ∧ do_exit invoked | `PidNs::reboot` |

### Layer 4: Verus/Creusot functional

`Per-CLONE_NEWPID at fork → create_pid_namespace → new init in child → zap_pid_ns_processes on init exit` semantic equivalence per `man 7 pid_namespaces` + `Documentation/admin-guide/namespaces/pid_namespaces.rst`. `setns(2)` semantics per `man 2 setns` (PID namespace is special: `setns` updates only `pid_for_children`, not active).

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

PID-namespace reinforcement:

- **Per-MAX_PID_NS_LEVEL (32) depth ceiling** — defense against per-recursive-clone stack/struct-pid explosion.
- **Per-UCOUNT_PID_NAMESPACES RLIMIT_NPROC slot** — defense against per-unprivileged ns flood.
- **Per-`in_userns(parent.user_ns, user_ns)` check** — defense against per-cross-user-ns pid-ns adoption.
- **Per-`pidns_is_ancestor` setns guard** — defense against per-pid-ns escape (cannot ascend).
- **Per-double-`ns_capable(CAP_SYS_ADMIN)` (new + cred)** — defense against per-cred-laundering setns.
- **Per-`disable_pid_allocation` before zap-scan** — defense against per-fork-race during ns teardown.
- **Per-`SIGCHLD = SIG_IGN` in zap_pid_ns_processes** — defense against per-zombie storm (auto-reap).
- **Per-`__fatal_signal_pending` check before SIGKILL** — defense against per-redundant kill flood.
- **Per-`SEND_SIG_PRIV` SIGKILL delivery** — defense against per-LSM-blocked kill in shutdown.
- **Per-`RCU-delayed pid_namespace free`** — defense against per-UAF on concurrent `task_active_pid_ns` readers.
- **Per-`destroy_pid_namespace_work` workqueue** — defense against per-locking inversion if destroy synchronously in put_pid_ns context.
- **Per-`reboot_pid_ns` rejects unsupported cmds** — defense against per-arbitrary signal injection via sys_reboot.
- **Per-`checkpoint_restore_ns_capable` for ns_last_pid write** — defense against per-pid-prediction attack.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/pid.c (`alloc_pid` / `free_pid` / `find_pid_ns` / per-`struct pid`) — sibling Tier-3
- kernel/nsproxy.c (per-task ns container, `copy_namespaces`, `switch_task_namespaces`)
- kernel/user_namespace.c (covered in `user-namespace.md`)
- kernel/utsname.c / ipc/namespace.c / fs/mount.c / kernel/cgroup/namespace.c (other ns types)
- fs/proc/namespaces.c — /proc/<pid>/ns/* file glue
- ns_common / ns_tree / nstree.c — generic namespace bookkeeping
- proc accounting (`acct_exit_ns`) — kernel/acct.c
- CRIU restore protocol details
- Implementation code
