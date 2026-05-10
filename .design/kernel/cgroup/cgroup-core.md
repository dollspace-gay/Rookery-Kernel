# Tier-3: kernel/cgroup/cgroup-core — cgroup framework (tree, css_set, attach/fork/exit)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - kernel/cgroup/cgroup.c
  - kernel/cgroup/cgroup-internal.h
  - include/linux/cgroup.h
  - include/linux/cgroup-defs.h
-->

## Summary
Tier-3 design for the cgroup core framework: `struct cgroup` tree management, `struct cgroup_subsys` registration, `struct css_set` task→cgroup association, attach/fork/exit hooks, the cgroupfs filesystem (built atop kernfs), and the per-controller subtree-control + threaded-mode handling.

This is the layer every cgroup controller (cpu/memory/cpuset/io/pids/etc.) plugs into. Owns the mandatory `models/cgroup/attach_safety.tla` TLA+ model.

Sub-tier-3 of `kernel/cgroup/00-overview.md`. Pairs with `fs/sysfs/kernfs.md` (underlying filesystem layer), `kernel/cgroup/rstat.md` (recursive statistics), per-controller Tier-3s.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| cgroup core: tree, css_set, attach/fork/exit, cgroupfs, subtree_control | `kernel/cgroup/cgroup.c` |
| Internal types | `kernel/cgroup/cgroup-internal.h` |
| Public API | `include/linux/cgroup.h`, `include/linux/cgroup-defs.h` |

## Compatibility contract

### `struct cgroup_root` — per-mount-instance

```c
struct cgroup_root {
    struct kernfs_root *kf_root;
    unsigned int subsys_mask;
    int hierarchy_id;
    struct cgroup cgrp;
    struct cgroup *cgrp_ancestor_storage[CGROUP_ANCESTOR_STORAGE_SIZE];
    atomic_t nr_cgrps;
    struct list_head root_list;
    struct rcu_head rcu;
    unsigned int flags;
    char release_agent_path[PATH_MAX];
    char name[MAX_CGROUP_ROOT_NAMELEN];
};
```

cgroup-v2 has a single root (the unified hierarchy); cgroup-v1 has one root per per-controller hierarchy. Identical layout.

### `struct cgroup` (per-cgroup state)

```c
struct cgroup {
    struct cgroup_subsys_state self;
    unsigned long flags;
    int level;
    int max_depth;
    int nr_descendants;
    int nr_dying_descendants;
    int max_descendants;
    int nr_populated_csets;
    int nr_populated_domain_children;
    int nr_populated_threaded_children;
    int nr_threaded_children;
    struct kernfs_node *kn;
    struct cgroup_file procs_file;
    struct cgroup_file events_file;
    struct cgroup_file psi_files[NR_PSI_RESOURCES];
    u16 subtree_control;
    u16 subtree_ss_mask;
    u16 old_subtree_control;
    u16 old_subtree_ss_mask;
    struct cgroup_subsys_state __rcu *subsys[CGROUP_SUBSYS_COUNT];
    struct cgroup_root *root;
    struct list_head cset_links;
    struct list_head e_csets[CGROUP_SUBSYS_COUNT];
    struct cgroup *dom_cgrp;
    struct cgroup *old_dom_cgrp;
    struct cgroup_rstat_cpu __percpu *rstat_cpu;
    struct cgroup_rstat_base_cpu __percpu *rstat_base_cpu;
    struct list_head rstat_css_list;
    struct cgroup *anchor;
    struct list_head pidlists;
    struct mutex pidlist_mutex;
    wait_queue_head_t offline_waitq;
    struct work_struct release_agent_work;
    struct psi_group *psi;
    struct cgroup_bpf bpf;
    struct cgroup_freezer_state freezer;
    struct bpf_local_storage __rcu *bpf_cgrp_storage;
    /* etc. */
};
```

Layout-equivalent for first cache-line.

### `struct cgroup_subsys_state` (per-controller per-cgroup)

```c
struct cgroup_subsys_state {
    struct cgroup *cgroup;
    struct cgroup_subsys *ss;
    struct percpu_ref refcnt;
    struct list_head sibling;
    struct list_head children;
    struct list_head rstat_css_node;
    int id;
    unsigned int flags;
    u64 serial_nr;
    atomic_t online_cnt;
    struct work_struct destroy_work;
    struct rcu_work destroy_rwork;
    struct cgroup_subsys_state *parent;
};
```

Each controller's per-cgroup state struct embeds this as first member. Layout-equivalent.

### `struct css_set` (task↔cgroup association)

```c
struct css_set {
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
    refcount_t refcount;
    struct css_set *dom_cset;
    struct cgroup *dfl_cgrp;
    int nr_tasks;
    struct list_head tasks;
    struct list_head mg_tasks;
    struct list_head dying_tasks;
    struct list_head task_iters;
    struct list_head cgrp_links;
    struct list_head e_cset_node[CGROUP_SUBSYS_COUNT];
    struct list_head threaded_csets;
    struct list_head threaded_csets_node;
    struct hlist_node hlist;
    struct list_head mg_src_preload_node;
    struct list_head mg_dst_preload_node;
    struct list_head mg_node;
    struct cgroup *mg_src_cgrp;
    struct cgroup *mg_dst_cgrp;
    struct css_set *mg_dst_cset;
    bool dead;
    struct rcu_head rcu_head;
};
```

Multiple tasks share a css_set if they belong to same cgroups across all enabled controllers. Layout-equivalent.

### Tree mgmt: `cgroup_create / _destroy`

Each cgroup-v2 cgroup created via `mkdir <path>` in cgroupfs:
1. `cgroup_mkdir` → `cgroup_create`
2. Allocate `struct cgroup`
3. Per-controller `css_alloc` for each enabled-in-subtree controller
4. `cgroup_apply_control` to enable controllers
5. Insert into parent's children list
6. `kernfs_create_dir` to create on-disk dir

Removal via `rmdir <path>`:
1. `cgroup_rmdir` → `cgroup_destroy_locked`
2. Per-controller `css_offline` (mark css for removal)
3. `kernfs_remove` of cgroup's kernfs subtree
4. Wait for refcount drains
5. Per-controller `css_released` + `css_free`

Identical lifecycle.

### Attach: task migration to cgroup

Triggered by writing PID to `cgroup.procs`:
1. `cgroup_attach_task` finds target task
2. Build `struct cgroup_taskset` with task + new css_set
3. Per-controller `can_attach(tset)` — vetoes return -EXXX
4. If all `can_attach` succeed: per-controller `attach(tset)` actually moves
5. Per-controller `post_attach()` (no failure recovery here)
6. Update `task->cgroups` pointer to new css_set

If any `can_attach` fails: per-controller `cancel_attach(tset)` for all that succeeded (atomic rollback). 

Identical algorithm.

### Fork hooks: `cgroup_fork`, `cgroup_can_fork`, `cgroup_post_fork`

When a task is forked:
1. `cgroup_can_fork(parent, css_set, flags)` — per-controller `can_fork` veto
2. On clone success: `cgroup_post_fork(child)` — per-controller `fork(child)` adds child to parent's css_set
3. On clone failure: `cgroup_cancel_fork(parent, css_set)` — per-controller `cancel_fork`

Per-controller `pids` uses can_fork to veto fork at `pids.max` cap.

### Exit hooks: `cgroup_exit`, `cgroup_free`

When a task exits:
1. `cgroup_exit(task)` — per-controller `exit(task)` — controllers update accounting
2. After `release_task` final-cleanup: `cgroup_release(task)` + `cgroup_free(task)` (drop css_set ref)

### `cgroup.subtree_control` mechanism

Writing `+memory` to a cgroup's `cgroup.subtree_control` enables memory controller for child cgroups (not for self). Allows hierarchical resource enabling without requiring all-or-nothing per controller across whole tree.

`cgroup_apply_control` walks the tree applying enable/disable to all affected cgroups + per-controller css_alloc/free.

Identical algorithm.

### Threaded subtrees (cgroup-v2)

`cgroup.type = threaded` puts a cgroup into thread-mode (per-thread instead of per-process granularity). Useful for thread-pool resource accounting. `cgroup.threads` file shows TIDs. Thread-mode cgroups must have a domain cgroup as root; mixing process + thread mode is forbidden (validated at attach).

### Per-cgroup BPF programs (`cgroup_bpf`)

Per-cgroup BPF program attachments (BPF_CGROUP_INET_INGRESS, BPF_CGROUP_INET_EGRESS, BPF_CGROUP_DEVICE, BPF_CGROUP_SOCK, etc.) — used by Cilium per-cgroup network policy + Tetragon for security observability. Per-cgroup `cgroup_bpf` field tracks per-attach programs.

### `cgroup.kill`

Write `1` sends SIGKILL to all tasks in cgroup + descendants (recursive). Used by container runtimes for fast container teardown.

### `cgroup.freeze` (cgroup-v2)

Write `1` freezes all tasks in cgroup; `0` thaws. Per-cgroup freezer state machine (cross-ref `kernel/cgroup/freezer.md`).

### `cgroup.events` + `populated`

Read returns:
```
populated 0|1
frozen 0|1
```

`populated 0` ⇔ no live tasks in cgroup or descendants. Used by systemd to wait for cgroup-empty event (service teardown completed).

`poll(EPOLLPRI)` notifies on change.

## Requirements

- REQ-1: `struct cgroup_root` + `struct cgroup` + `struct cgroup_subsys_state` + `struct css_set` first-cache-line layouts.
- REQ-2: Per-cgroup tree mgmt: cgroup_create/destroy lifecycle identical.
- REQ-3: Attach: cgroup_attach_task with can_attach/attach/cancel_attach atomic-with-rollback semantics; post_attach fire-and-forget.
- REQ-4: Fork: cgroup_can_fork → cgroup_post_fork or cgroup_cancel_fork pattern.
- REQ-5: Exit: cgroup_exit + cgroup_release/cgroup_free; per-controller exit hook.
- REQ-6: `cgroup.subtree_control` enable/disable per-controller for descendants only; cgroup_apply_control walks tree.
- REQ-7: Threaded subtrees: `cgroup.type = threaded` + `cgroup.threads` per-thread granularity; mixing forbidden.
- REQ-8: Per-cgroup BPF program attach/detach (BPF_CGROUP_*); per-cgroup `cgroup_bpf` state.
- REQ-9: `cgroup.kill` recursive SIGKILL; `cgroup.freeze` per-cgroup freeze; `cgroup.events` populated + frozen with EPOLLPRI notification.
- REQ-10: cgroupfs built atop kernfs; per-instance `cgroup_root` corresponds to `kernfs_root`.
- REQ-11: TLA+ model `models/cgroup/attach_safety.tla` (mandatory per `kernel/cgroup/00-overview.md` Layer 2) — proves: attach atomicity (can_attach failure path rolls back consistently); concurrent attaches don't deadlock.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct cgroup` + `cgroup_subsys_state` + `css_set` byte-identical first cache-line. (covers REQ-1)
- [ ] AC-2: `mkdir /sys/fs/cgroup/test_cg` test: cgroup created; standard files visible (cgroup.procs/events/etc.); `rmdir` removes. (covers REQ-2)
- [ ] AC-3: Attach test: `echo $$ > /sys/fs/cgroup/test_cg/cgroup.procs`; subsequent `cat /proc/self/cgroup` shows test_cg path. (covers REQ-3)
- [ ] AC-4: can_attach veto test: pids controller with pids.max=0; attach 1st task fails with -EAGAIN; per-controller can_attach properly cancelled. (covers REQ-3)
- [ ] AC-5: Fork test: parent in cgroup A; fork; child auto-attached to A; pids controller's fork hook accounts. (covers REQ-4)
- [ ] AC-6: Exit test: task in cgroup A exits; cgroup A's nr_populated decrements; events file's populated transitions 1→0 if last task. (covers REQ-5, REQ-9)
- [ ] AC-7: subtree_control test: `echo +memory > parent/cgroup.subtree_control`; child cgroups now have memory.* files. (covers REQ-6)
- [ ] AC-8: Threaded test: `echo threaded > foo/cgroup.type`; `cat foo/cgroup.threads`; per-thread entries appear. (covers REQ-7)
- [ ] AC-9: BPF attach test: `bpftool cgroup attach <prog> /sys/fs/cgroup/test_cg`; per-cgroup BPF list shows attached. (covers REQ-8)
- [ ] AC-10: cgroup.kill test: `echo 1 > /sys/fs/cgroup/test_cg/cgroup.kill`; all tasks in test_cg + descendants receive SIGKILL. (covers REQ-9)
- [ ] AC-11: cgroup.freeze test: `echo 1 > .../cgroup.freeze`; tasks in cgroup are frozen (state D / TASK_FROZEN); `echo 0 >` thaws. (covers REQ-9)
- [ ] AC-12: TLA+ `models/cgroup/attach_safety.tla` proves: attach atomicity (can_attach failure → all prior cancel_attached); concurrent attaches don't deadlock. (covers REQ-11)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::cgroup::core::Cgroup` — `struct cgroup` core
- `kernel::cgroup::core::Root` — `struct cgroup_root`
- `kernel::cgroup::core::SubsysState` — `struct cgroup_subsys_state`
- `kernel::cgroup::core::Subsys` — `struct cgroup_subsys` registration
- `kernel::cgroup::core::CssSet` — `struct css_set`
- `kernel::cgroup::core::Tree` — tree mgmt (create/destroy)
- `kernel::cgroup::core::Attach` — cgroup_attach_task + can_attach/attach/cancel_attach
- `kernel::cgroup::core::Fork` — cgroup_can_fork + cgroup_post_fork + cgroup_cancel_fork
- `kernel::cgroup::core::Exit` — cgroup_exit + cgroup_release
- `kernel::cgroup::core::SubtreeControl` — cgroup.subtree_control + cgroup_apply_control
- `kernel::cgroup::core::Threaded` — threaded subtrees
- `kernel::cgroup::core::BpfAttach` — per-cgroup BPF program attach
- `kernel::cgroup::core::Kill` — cgroup.kill recursive SIGKILL
- `kernel::cgroup::core::Events` — cgroup.events populated/frozen + EPOLLPRI

### Locking and concurrency

- **`cgroup_mutex`** (mutex): per-system tree mutator
- **`css_set_lock`** (spinlock): per-system css_set hash + ref mutator
- **Per-cgroup `kn->mutex`** (kernfs): per-cgroup file mutator
- **Per-css `percpu_ref`**: per-controller refcount
- **RCU**: per-css lookup hot path RCU-side; tree walk via `cgroup_for_each_*`

### Error handling

- `Err(EAGAIN)` — can_attach veto (e.g., pids.max)
- `Err(EINVAL)` — bad attach request (mixing thread + process modes)
- `Err(EBUSY)` — rmdir on populated cgroup
- `Err(EPERM)` — non-CAP_SYS_RESOURCE
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Tree create/destroy under cgroup_mutex (refcount sound) | `kani::proofs::kernel::cgroup::core::tree_safety` |
| Attach atomicity (can_attach veto + cancel_attach rollback) | `kani::proofs::kernel::cgroup::core::attach_safety` |
| Fork hook integration (per-task css_set ref-acquire under fork lock) | `kani::proofs::kernel::cgroup::core::fork_safety` |
| Exit hook integration (per-controller exit + css_set ref-release) | `kani::proofs::kernel::cgroup::core::exit_safety` |
| cgroup.kill recursive walk | `kani::proofs::kernel::cgroup::core::kill_safety` |

### Layer 2: TLA+ models

- `models/cgroup/attach_safety.tla` (mandatory per `kernel/cgroup/00-overview.md`) — proves attach atomicity. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-cgroup tree | every non-root cgroup has parent; tree depth ≤ max_depth; total descendants ≤ max_descendants | `kani::proofs::kernel::cgroup::core::tree_invariants` |
| Per-css_set | every css_set has correct subsys[] for cgroup-set; refcount = unique tasks count | `kani::proofs::kernel::cgroup::core::cset_invariants` |
| Per-css refcount | refcount > 0 between css_alloc and css_free | `kani::proofs::kernel::cgroup::core::css_ref_invariants` |

### Layer 4: Functional correctness (declared in `kernel/cgroup/00-overview.md` Layer 4)

- **cgroup attach atomicity theorem** via Verus — proves: per-task attach is atomic across all enabled controllers; on per-controller can_attach failure, all prior succeeded can_attach are cancel_attach'd. Owned here.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-cgroup + per-css + per-css_set refcounts use `Refcount` (saturating) + percpu_ref for css | § Mandatory |
| **AUTOSLAB** | per-cgroup + per-css_set slab caches | § Mandatory |
| **CONSTIFY** | per-controller `cgroup_subsys` instances `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed cgroup + css_set state cleared (carry per-cgroup name + LSM secctx + per-controller config) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, MEMORY_SANITIZE**: see above
- **USERCOPY**: per-cgroup file read/write uses kernfs bound-checked accessors (cross-ref `fs/sysfs/kernfs.md`)
- **SIZE_OVERFLOW**: per-cgroup id + per-tree depth arithmetic uses checked operators
- **KERNEXEC**: per-controller dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook `security_cgroup_attach` for per-task migration.
- LSM hook `security_capable(CAP_SYS_RESOURCE)` for resource-cap mutations.
- Default useful GR-RBAC policy: deny `cgroup.kill` writes outside gradm-marked `container_admin` role; deny per-controller `<resource>.max` writes that would expand a child's limit beyond parent's.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — cgroup core framework exhaustively specified by upstream + cgroup-v2 documentation + container-runtime test corpus)

## Out of Scope

- Per-controller implementations (cross-ref individual Tier-3s — `cpuset.md` / `freezer.md` / `pids.md` / `mm/memcontrol.md` / `block/blk-cgroup.md` / etc.)
- rstat (cross-ref `kernel/cgroup/rstat.md`)
- cgroup-v1 legacy (cross-ref `kernel/cgroup/cgroup-v1.md`)
- 32-bit-only paths
- Implementation code
