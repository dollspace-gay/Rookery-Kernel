---
title: "Tier-2: kernel/cgroup — control groups (cgroup-v1 + cgroup-v2 unified)"
tags: ["design-doc", "tier-2", "kernel", "cgroup"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for control groups (cgroup) — Linux's resource-control + accounting + isolation framework. The backbone of every modern container runtime + systemd's resource-isolated services + Kubernetes' pod-level resource enforcement. Two coexisting models:

- **cgroup-v1**: per-controller separate hierarchies; mounted as `/sys/fs/cgroup/<controller>/...`
- **cgroup-v2 (unified)**: single hierarchy with all controllers; mounted as `/sys/fs/cgroup/...`; modern + recommended; default in systemd-managed distros since v243+

Each cgroup is a directory in `/sys/fs/cgroup/...`; tasks added by writing PID to `cgroup.procs` (or `cgroup.threads` for thread-mode subtree). Per-controller files (`memory.max`, `cpu.max`, `pids.max`, etc.) configure resource limits.

13 in-tree controllers:
- **CPU** (`cpu`): CPU time accounting + scheduling priority + bandwidth control (cross-ref `kernel/sched/cfs.md` cgroup-aware bandwidth)
- **Memory** (`memory`/`memcontrol`): per-cgroup memory cap + accounting + pressure tracking + OOM kill (`mm/memcontrol.c`)
- **CPUSet** (`cpuset`): per-cgroup CPU affinity + NUMA memory node affinity (`cpuset.c`)
- **Block I/O** (`io` v2 / `blkio` v1): per-cgroup I/O accounting + throttling (`blk-cgroup.c`)
- **PIDs** (`pids`): per-cgroup PID count cap (`pids.c`)
- **Devices** (`devices` legacy v1; replaced by BPF in v2)
- **Freezer** (`freezer`): per-cgroup task suspend/resume (`legacy_freezer.c` v1, `freezer.c` v2)
- **HugeTLB** (`hugetlb`): per-cgroup huge-page count cap
- **RDMA** (`rdma`): per-cgroup RDMA hca handles cap (`rdma.c`)
- **Misc** (`misc`): generic per-resource cap (`misc.c`)
- **DMEM** (`dmem`): per-cgroup device memory accounting (`dmem.c`)
- **Net classifier** (`net_cls` v1 only): per-cgroup classid for tc rules
- **Net priority** (`net_prio` v1 only): per-cgroup per-iface priority

cgroup-v2 deprecates `net_cls` + `net_prio` + `devices` (replaced by BPF + `cgroup.controllers` API).

Per-cgroup recursive statistics via `rstat` (`rstat.c`) — flush-only-when-needed pattern, cross-cpu aggregation.

Per-cgroup namespace (`namespace.c`) — `unshare(CLONE_NEWCGROUP)` creates per-task cgroup-namespace such that `/proc/<pid>/cgroup` shows root-relative paths.

Sub-tier-2 of `kernel/00-overview.md`. Built atop kernfs (cross-ref `fs/sysfs/kernfs.md`).

### Out of Scope

- Per-Tier-3 child docs (Phase C continuation)
- Per-controller deep-dives (cross-ref individual Tier-3s)
- 32-bit-only paths
- Implementation code

### scope

This Tier-2 governs all of `/home/doll/linux-src/kernel/cgroup/` (~14 source files), the per-controller files in `mm/memcontrol.c` + `block/blk-cgroup.c`, plus public API + UAPI headers.

### compatibility contract — outline

### Mount semantics

cgroup-v2: `mount -t cgroup2 cgroup2 /sys/fs/cgroup` (the default since systemd v243). Single unified hierarchy.

cgroup-v1: per-controller mounts:
```
mount -t cgroup -o cpu cgroup /sys/fs/cgroup/cpu
mount -t cgroup -o memory cgroup /sys/fs/cgroup/memory
... (one per controller)
```

Hybrid mode: cgroup-v2 mount with v1 controller fallback (`memory_recursiveprot`, `nsdelegate`, `memory_localevents`, `memory_hugetlb_accounting` mount options).

Identical mount semantics + options.

### Per-cgroup file conventions

Common files in every cgroup directory:
- `cgroup.procs` — task PIDs in this cgroup (writable: add task to cgroup; read: list tasks)
- `cgroup.threads` — thread IDs (for thread-mode subtree)
- `cgroup.controllers` — read-only: list of available controllers
- `cgroup.subtree_control` — writable: enabled controllers (with +/- prefix)
- `cgroup.events` — `populated 0/1 frozen 0/1` (read + poll for state changes)
- `cgroup.freeze` — write 1 to freeze, 0 to unfreeze
- `cgroup.kill` — write 1 to send SIGKILL to all in-cgroup tasks
- `cgroup.max.depth` — cap on subtree depth
- `cgroup.max.descendants` — cap on descendant count
- `cgroup.stat` — `nr_descendants nr_dying_descendants` etc.
- `cgroup.type` — `domain`, `domain threaded`, `domain invalid`, `threaded`
- `cgroup.pressure` — pressure-information per cgroup (PSI)
- `cgroup.idle` — write 1 to mark cgroup idle (boost OOM-priority)

Plus per-controller files (memory.max, cpu.max, etc.).

### Per-controller files (cgroup-v2)

| Controller | Files (subset) |
|---|---|
| `memory` | `memory.max`, `memory.high`, `memory.low`, `memory.min`, `memory.current`, `memory.peak`, `memory.swap.max`, `memory.events`, `memory.stat`, `memory.zswap.max`, `memory.numa_stat` |
| `cpu` | `cpu.max`, `cpu.weight`, `cpu.weight.nice`, `cpu.stat`, `cpu.idle`, `cpu.uclamp.max`, `cpu.uclamp.min`, `cpu.pressure` |
| `cpuset` | `cpuset.cpus`, `cpuset.mems`, `cpuset.cpus.effective`, `cpuset.mems.effective`, `cpuset.cpus.partition`, `cpuset.cpus.exclusive` |
| `io` | `io.max`, `io.weight`, `io.bfq.weight`, `io.cost.qos`, `io.stat`, `io.pressure`, `io.latency` |
| `pids` | `pids.max`, `pids.current`, `pids.events`, `pids.peak` |
| `hugetlb` | `hugetlb.<size>.max`, `hugetlb.<size>.current`, `hugetlb.<size>.events` |
| `rdma` | `rdma.max`, `rdma.current` |
| `misc` | `misc.max`, `misc.current`, `misc.events` |
| `dmem` | `dmem.max`, `dmem.current` |

Format byte-identical so existing distro / container / k8s tooling parses correctly.

### `struct cgroup` core layout

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
    /* psi etc. */
};
```

Layout-equivalent for first cache-line.

### `struct cgroup_subsys_state` (per-controller per-cgroup state)

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

Per-controller embeds this as first member (e.g., `struct mem_cgroup` starts with `cgroup_subsys_state`).

### `struct cgroup_subsys` (per-controller registration)

```c
struct cgroup_subsys {
    struct cgroup_subsys_state *(*css_alloc)(struct cgroup_subsys_state *parent_css);
    int (*css_online)(struct cgroup_subsys_state *css);
    void (*css_offline)(struct cgroup_subsys_state *css);
    void (*css_released)(struct cgroup_subsys_state *css);
    void (*css_free)(struct cgroup_subsys_state *css);
    void (*css_reset)(struct cgroup_subsys_state *css);
    void (*css_rstat_flush)(struct cgroup_subsys_state *css, int cpu);
    int (*css_extra_stat_show)(struct seq_file *seq, struct cgroup_subsys_state *css);
    int (*css_local_stat_show)(struct seq_file *seq, struct cgroup_subsys_state *css);
    int (*can_attach)(struct cgroup_taskset *tset);
    void (*cancel_attach)(struct cgroup_taskset *tset);
    void (*attach)(struct cgroup_taskset *tset);
    void (*post_attach)(void);
    int (*can_fork)(struct task_struct *task, struct css_set *cset);
    void (*cancel_fork)(struct task_struct *task, struct css_set *cset);
    void (*fork)(struct task_struct *task);
    void (*exit)(struct task_struct *task);
    void (*release)(struct task_struct *task);
    void (*bind)(struct cgroup_subsys_state *root_css);
    bool early_init:1;
    bool implicit_on_dfl:1;
    bool threaded:1;
    int id;
    const char *name;
    const char *legacy_name;
    struct cgroup_root *root;
    struct idr css_idr;
    struct list_head cfts;
    struct cftype *dfl_cftypes;
    struct cftype *legacy_cftypes;
    unsigned int depends_on;
};
```

Per-controller registers via `cgroup_subsys_register`. Identical layout.

### css_set + task_css_set association

`struct css_set` represents a set of `cgroup_subsys_state`s — one per controller. Multiple tasks share a css_set if they're in the same cgroup-set (same cgroups for all controllers). Tasks reference css_set via `task->cgroups`.

Pattern: minimize task→cgroup pointer indirection by sharing css_set across tasks.

### rstat — recursive statistics

`rstat.c` provides flush-on-read recursive statistics: per-cpu per-cgroup updates accumulate locally; on read, kernel does a tree-walk flush from the queried cgroup up to root, aggregating per-cpu values.

Identical algorithm.

### Per-cgroup PSI (Pressure Stall Information)

Per-cgroup `cpu.pressure`, `memory.pressure`, `io.pressure` files report Pressure-Stall-Information time-percentages (cross-ref `kernel/sched/psi.md` Tier-3 if added). PSI is the cornerstone of cgroup-aware OOM kill + autoscaling.

### `cgroup.events`

Read returns:
```
populated 0|1
frozen 0|1
```

Plus `poll(EPOLLPRI)` for state changes. Used by systemd to wait for cgroup-empty events (service teardown).

### tier-3 docs governed by this tier-2

| Tier-3 doc | Scope |
|---|---|
| `kernel/cgroup/cgroup-core.md` | cgroup framework: cgroup tree, css_set, attach/fork/exit (`cgroup.c`) |
| `kernel/cgroup/cgroup-v1.md` | v1 legacy compatibility (`cgroup-v1.c`) |
| `kernel/cgroup/rstat.md` | recursive statistics (`rstat.c`) |
| `kernel/cgroup/cpuset.md` | cpuset controller (`cpuset.c` + `cpuset-v1.c`) |
| `kernel/cgroup/freezer.md` | freezer controller (`freezer.c` + `legacy_freezer.c`) |
| `kernel/cgroup/pids.md` | pids controller (`pids.c`) |
| `kernel/cgroup/misc.md` | misc + rdma + dmem (`misc.c` + `rdma.c` + `dmem.c`) |
| `kernel/cgroup/namespace.md` | cgroup namespace (`namespace.c`) |
| `mm/memcontrol.md` | memory controller (`memcontrol.c` — sub-tier-3 of `mm/`) |
| `block/blk-cgroup.md` | I/O controller (`blk-cgroup.c` — sub-tier-3 of `block/`) |

### compatibility outline (top-level)

- REQ-O1: cgroup-v2 unified hierarchy + cgroup-v1 per-controller hierarchies both supported.
- REQ-O2: All standard cgroup-v2 files (cgroup.procs / threads / controllers / subtree_control / events / freeze / kill / max.depth / max.descendants / stat / type / pressure / idle) byte-identical.
- REQ-O3: 13 in-tree controllers (cpu / memory / cpuset / io / pids / devices [v1] / freezer / hugetlb / rdma / misc / dmem / net_cls [v1] / net_prio [v1]) implemented identically.
- REQ-O4: `struct cgroup` + `struct cgroup_subsys_state` + `struct cgroup_subsys` + `struct css_set` first-cache-line layouts.
- REQ-O5: rstat tree-walk flush-on-read for per-cgroup statistics.
- REQ-O6: PSI integration: per-cgroup cpu.pressure / memory.pressure / io.pressure files.
- REQ-O7: cgroup namespace via `CLONE_NEWCGROUP` flag in unshare/clone.
- REQ-O8: Per-Tier-3 child documents define per-controller requirements.
- REQ-O9: TLA+ models declared at this Tier-2 (see § Verification — Layer 2 below).
- REQ-O10: Hardening: row-1 features applied per `00-security-principles.md`.

### acceptance criteria (top-level)

- [ ] AC-O1: `mount -t cgroup2 ... /sys/fs/cgroup` succeeds; standard files visible. (covers REQ-O1, REQ-O2)
- [ ] AC-O2: All controllers (per the table) attachable + functional. (covers REQ-O3)
- [ ] AC-O3: `pahole struct cgroup` etc. byte-identical first cache-line. (covers REQ-O4)
- [ ] AC-O4: rstat test: per-cpu memory.stat updates aggregate correctly on read. (covers REQ-O5)
- [ ] AC-O5: PSI test: per-cgroup cpu.pressure shows non-zero values under load. (covers REQ-O6)
- [ ] AC-O6: `unshare --cgroup` test: child sees `/proc/self/cgroup` rooted at unshared cgroup. (covers REQ-O7)
- [ ] AC-O7: Per-Tier-3 ACs cumulatively cover cgroup ABI. (meta-AC)
- [ ] AC-O8: TLA+ models per § Verification all pass. (covers REQ-O9)
- [ ] AC-O9: Hardening section per Tier-3 child docs is non-empty. (covers REQ-O10)

### architecture (top-level)

Each Tier-3 child doc declares its own Rust module organization. Shared abstractions:

- `kernel::cgroup::Cgroup` — `struct cgroup` core
- `kernel::cgroup::Subsys` — `struct cgroup_subsys` per-controller registration
- `kernel::cgroup::SubsysState` — `struct cgroup_subsys_state`
- `kernel::cgroup::CssSet` — `struct css_set`
- `kernel::cgroup::Hierarchy` — cgroup tree mgmt
- `kernel::cgroup::Attach` — attach/fork/exit hooks
- `kernel::cgroup::Rstat` — recursive statistics
- `kernel::cgroup::Psi` — pressure-information integration
- `kernel::cgroup::Namespace` — CLONE_NEWCGROUP

### verification (top-level)

### Layer 1: Kani SAFETY proofs

Per Tier-3.

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/cgroup/attach_safety.tla` | `kernel/cgroup/cgroup-core.md` (proves: attach hooks (can_attach + attach + cancel_attach) execute atomically per-task; on cancel, all per-controller state rolled back) |
| `models/cgroup/rstat_consistency.tla` | `kernel/cgroup/rstat.md` (proves: rstat tree-walk produces consistent aggregate values; concurrent updates don't lose increments; flush-on-read is safe under concurrent writers) |

### Layer 3: invariant harnesses

Per Tier-3.

### Layer 4: functional correctness (opt-in)

- **cgroup attach atomicity theorem** via Verus — proves: per-task attach is atomic across all enabled controllers; on per-controller can_attach failure, all prior can_attach are cancel_attach'd.

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-cgroup + per-css + per-css_set refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-cgroup + per-css + per-css_set slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed cgroup state cleared (carries per-cgroup name + LSM secctx + per-controller config) | § Default-on configurable off |
| **CONSTIFY** | per-controller `cgroup_subsys` instances `static const` | § Mandatory |

### Row-1 features consumed by this Tier-2

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, CONSTIFY**: see above
- **USERCOPY**: per-cgroup file read/write uses kernfs bound-checked accessors
- **SIZE_OVERFLOW**: per-cgroup id + per-cset hash arithmetic uses checked operators
- **KERNEXEC**: per-controller dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hooks: `security_inode_permission` + `security_file_permission` for per-cgroup file access; `security_capable(CAP_SYS_RESOURCE)` for resource cap mutations.
- Default useful GR-RBAC policy: deny per-cgroup `cgroup.kill` writes outside gradm-marked `container_admin`; deny `memory.max` writes that would expand a child cgroup's limit beyond parent's.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

