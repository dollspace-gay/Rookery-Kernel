---
title: "Tier-3: fs/proc/proc-generic — generic procfs framework (proc_dir_entry, inodes, root)"
tags: ["design-doc", "tier-3", "fs", "procfs"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the generic procfs framework that all per-feature Tier-3s build on. Covers:

- **`struct proc_dir_entry`** — the per-entry descriptor representing each procfs file/directory
- **`proc_create_*` family** — registration entry-points used by every kernel module that exposes a procfs file
- **`struct proc_inode`** — per-procfs-inode private state
- **`procfs_root` superblock setup** — `proc_fill_super`, mount options (hidepid/gid/subset)
- **Per-pid/per-tgid lookup** — fast-path resolution of `/proc/<pid>/...`
- **`proc_namespace.c`** — per-mount-namespace `/proc/mounts`/`mountinfo`/`mountstats` rendering
- **String-formatting + small utility helpers** — `util.c`

This is the layer that every other procfs Tier-3 (proc-task, proc-system, proc-sysctl, proc-net, proc-namespaces, proc-fd) consumes.

Sub-tier-3 of `fs/proc/00-overview.md`.

### Requirements

- REQ-1: `struct proc_dir_entry` first-cache-line layout-equivalent.
- REQ-2: `struct proc_ops` modern-per-file ops layout-byte-identical; PROC_ENTRY_PERMANENT + FORCE_LOOKUP flags.
- REQ-3: `proc_create_*` family (proc_create / _data / _seq / _seq_data / _single / _single_data / _mount_point) signatures + behavior identical.
- REQ-4: `proc_mkdir / _mode / _symlink` directory + symlink creation identical.
- REQ-5: `proc_remove / remove_proc_entry / remove_proc_subtree` removal semantics + RCU-defer free identical.
- REQ-6: `procfs_root` superblock setup + mount options (hidepid / gid / subset) byte-identical.
- REQ-7: Per-pid + per-tgid lookup chain identical.
- REQ-8: `struct proc_inode` first-cache-line layout-equivalent.
- REQ-9: `/proc/mounts` + `/proc/mountinfo` + `/proc/mountstats` per-task format byte-identical.
- REQ-10: Top-level symlinks `/proc/self`, `/proc/thread-self`, `/proc/mounts`, `/proc/net` identical resolution.
- REQ-11: Per-PDE refcount + RCU lifecycle: `proc_remove` waits for refcnt-drop; concurrent in-flight readers see consistent view.
- REQ-12: `proc_create_mount_point` mount-override-friendly directory creation.
- REQ-13: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct proc_dir_entry` + `struct proc_ops` + `struct proc_inode` byte-identical first cache-line. (covers REQ-1, REQ-2, REQ-8)
- [ ] AC-2: `proc_create_data` test: synthetic kernel-module registers a `/proc/test_entry` with read callback; `cat /proc/test_entry` returns expected content. (covers REQ-3)
- [ ] AC-3: `proc_create_seq` test: register seq_ops; `cat` walks seq via start/next/stop/show. (covers REQ-3)
- [ ] AC-4: `proc_mkdir + proc_symlink` test: synthetic module creates `/proc/foo/`, `/proc/foo/bar -> baz`; readlink confirms. (covers REQ-4)
- [ ] AC-5: `proc_remove` test: synthetic test creates entry; concurrent reader holding a copy + remove → reader completes existing read; subsequent open returns ENOENT. (covers REQ-5, REQ-11)
- [ ] AC-6: Mount options test: `mount -t proc -o hidepid=2,gid=1000 proc /proc`; non-root non-gid-1000 user sees only their own pid entries. (covers REQ-6)
- [ ] AC-7: `subset=pid` test: `mount -t proc -o subset=pid proc /proc`; system-wide entries (cpuinfo/meminfo) not visible. (covers REQ-6)
- [ ] AC-8: Per-pid lookup test: `cat /proc/$$/cmdline` resolves correctly via proc_pid_lookup → pid→task→cmdline. (covers REQ-7)
- [ ] AC-9: `/proc/mounts` byte-identical to `/etc/mtab`-style format. (covers REQ-9)
- [ ] AC-10: `/proc/mountinfo` byte-identical to upstream's per-mount extended format. (covers REQ-9)
- [ ] AC-11: Symlink resolution: `readlink /proc/self`, `/proc/thread-self`, `/proc/mounts`, `/proc/net` return expected targets. (covers REQ-10)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-13)

### Architecture

### Rust module organization

- `kernel::fs::proc::generic::ProcDirEntry` — `struct proc_dir_entry` wrapper
- `kernel::fs::proc::generic::ProcOps` — `struct proc_ops` wrapper
- `kernel::fs::proc::generic::Create` — `proc_create_*` registration family
- `kernel::fs::proc::generic::Mkdir` — `proc_mkdir` + `proc_mkdir_mode`
- `kernel::fs::proc::generic::Symlink` — `proc_symlink`
- `kernel::fs::proc::generic::Remove` — `proc_remove` + RCU-defer
- `kernel::fs::proc::generic::SeqAdapter` — bridges `proc_create_seq` → `seq_file`
- `kernel::fs::proc::inode::ProcInode` — `struct proc_inode`
- `kernel::fs::proc::inode::Lifecycle` — alloc/free/evict
- `kernel::fs::proc::root::FillSuper` — `proc_fill_super`
- `kernel::fs::proc::root::MountOptions` — hidepid/gid/subset
- `kernel::fs::proc::root::PidLookup` — `/proc/<pid>` resolution
- `kernel::fs::proc::namespace::ProcMounts` — `/proc/mounts`
- `kernel::fs::proc::namespace::ProcMountinfo` — `/proc/mountinfo`
- `kernel::fs::proc::namespace::ProcMountstats` — `/proc/mountstats`
- `kernel::fs::proc::util::*` — string helpers

### Locking and concurrency

- **Per-parent `parent->subdir_lock`** (rwsem): per-directory subdirectory tree mutator
- **Per-`proc_dir_entry` `refcnt`**: refcount-protected; RCU-defer free on `proc_remove`
- **`pde_openers` list + `pde_unload_lock`**: tracks open files for entries being removed
- **Per-mount super_block lock**: standard VFS lock

### Error handling

- `Err(EEXIST)` — duplicate name in parent
- `Err(ENOENT)` — entry not found
- `Err(EBUSY)` — entry has open files (proc_remove waits)
- `Err(EACCES)` — permission denied
- `Err(ENOMEM)` — alloc fail

### Out of Scope

- Per-feature procfs Tier-3s (cross-ref `proc-task.md`, `proc-system.md`, `proc-sysctl.md`, `proc-net.md`, `proc-namespaces.md`, `proc-fd.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Generic procfs framework: proc_create_*, proc_dir_entry, dir-walk | `fs/proc/generic.c` |
| procfs inode lifecycle | `fs/proc/inode.c` |
| Root mount + super_block setup + per-mount options | `fs/proc/root.c` |
| Small utilities | `fs/proc/util.c` |
| Internal types (struct proc_dir_entry, etc.) | `fs/proc/internal.h` |
| Per-mount-namespace mountinfo/mounts/mountstats | `fs/proc_namespace.c` |
| Public API | `include/linux/proc_fs.h` |

### compatibility contract

### `struct proc_dir_entry` layout

`fs/proc/internal.h`:
```c
struct proc_dir_entry {
    /* refcount-based with awaiters */
    atomic_t in_use;
    refcount_t refcnt;
    struct list_head pde_openers;
    spinlock_t pde_unload_lock;
    struct completion *pde_unload_completion;
    const struct proc_ops *proc_ops;
    const struct seq_operations *seq_ops;
    int (*single_show)(struct seq_file *, void *);
    proc_write_t write;
    void *data;
    unsigned int state_size;
    unsigned int low_ino;
    nlink_t nlink;
    kuid_t uid;
    kgid_t gid;
    loff_t size;
    const struct inode_operations *proc_iops;
    union {
        const struct seq_operations *seq_ops;
        int (*single_show)(struct seq_file *, void *);
    };
    proc_write_t write;
    void *data;
    unsigned int state_size;
    unsigned int namelen;
    struct rb_node subdir_node;
    char *name;
    umode_t mode;
    /* ... */
};
```

Layout-equivalent for first cache-line.

### `struct proc_ops` (modern per-file ops)

```c
struct proc_ops {
    unsigned int proc_flags;
    int (*proc_open)(struct inode *, struct file *);
    ssize_t (*proc_read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*proc_read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t (*proc_write)(struct file *, const char __user *, size_t, loff_t *);
    loff_t (*proc_lseek)(struct file *, loff_t, int);
    int (*proc_release)(struct inode *, struct file *);
    __poll_t (*proc_poll)(struct file *, struct poll_table_struct *);
    long (*proc_ioctl)(struct file *, unsigned int, unsigned long);
#ifdef CONFIG_COMPAT
    long (*proc_compat_ioctl)(struct file *, unsigned int, unsigned long);
#endif
    int (*proc_mmap)(struct file *, struct vm_area_struct *);
    unsigned long (*proc_get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
};
```

`proc_flags` includes `PROC_ENTRY_PERMANENT` (don't refcount-track) and `PROC_ENTRY_FORCE_LOOKUP` (always re-resolve).

Replaced legacy `struct file_operations` registration to keep procfs-specific helpers separate from VFS file ops. Identical layout.

### `proc_create_*` registration family

```c
struct proc_dir_entry *proc_create(const char *name, umode_t mode,
                                   struct proc_dir_entry *parent,
                                   const struct proc_ops *proc_ops);
struct proc_dir_entry *proc_create_data(const char *name, umode_t mode,
                                        struct proc_dir_entry *parent,
                                        const struct proc_ops *proc_ops, void *data);
struct proc_dir_entry *proc_create_seq(const char *name, umode_t mode,
                                       struct proc_dir_entry *parent,
                                       const struct seq_operations *ops);
struct proc_dir_entry *proc_create_seq_data(...);
struct proc_dir_entry *proc_create_single(const char *name, umode_t mode,
                                          struct proc_dir_entry *parent,
                                          int (*show)(struct seq_file *, void *));
struct proc_dir_entry *proc_create_single_data(...);
struct proc_dir_entry *proc_create_mount_point(const char *name);

struct proc_dir_entry *proc_mkdir(const char *name, struct proc_dir_entry *parent);
struct proc_dir_entry *proc_mkdir_mode(const char *name, umode_t mode, struct proc_dir_entry *parent);
struct proc_dir_entry *proc_symlink(const char *name, struct proc_dir_entry *parent, const char *dest);

void proc_remove(struct proc_dir_entry *de);
void remove_proc_entry(const char *name, struct proc_dir_entry *parent);
int  remove_proc_subtree(const char *name, struct proc_dir_entry *parent);
```

Variants per use case:
- `_seq` — multi-line seq_file from `seq_operations`
- `_single` — one-shot seq_file from a single `show` callback
- Plus `_data` variants accepting per-file private data

Identical signatures.

### Per-mount super_block + options

`fs/proc/root.c` implements `proc_fill_super`:
```c
static const struct super_operations proc_sops = {
    .alloc_inode    = proc_alloc_inode,
    .free_inode     = proc_free_inode,
    .drop_inode     = generic_delete_inode,
    .evict_inode    = proc_evict_inode,
    .statfs         = simple_statfs,
    .show_options   = proc_show_options,
};
```

Mount options:
- `hidepid=N` — 0 (all visible) | 1 (hide other-uid pid contents) | 2 (hide other-uid pid listings)
- `gid=N` — group bypassing hidepid restrictions
- `subset=pid` — only `/proc/<pid>` visible (no system-wide entries) — used by container runtimes

### Per-pid + per-tgid lookup

`fs/proc/base.c` (cross-ref `proc-task.md`) but the framework lookup is in `proc/root.c`. For `/proc/<N>` lookup:
1. Check if N is a valid pid in current pid-ns
2. If yes, resolve to `proc_pid_lookup` → returns per-pid directory inode
3. Otherwise, fall through to static-entry lookup

Identical lookup chain.

### `struct proc_inode` per-inode state

```c
struct proc_inode {
    struct pid *pid;                          /* for /proc/<pid>/ entries */
    unsigned int fd;                          /* for /proc/<pid>/fd/<n> entries */
    union proc_op op;                         /* per-file callback */
    struct proc_dir_entry *pde;               /* per-PDE entry */
    struct ctl_table_header *sysctl;          /* for /proc/sys/* entries */
    struct ctl_table *sysctl_entry;
    struct hlist_node sibling_inodes;
    const struct proc_ns_operations *ns_ops;  /* for /proc/<pid>/ns/* */
    struct inode vfs_inode;                   /* embedded VFS inode */
};
```

Layout-equivalent for first cache-line.

### `/proc/mounts` / `/proc/mountinfo` / `/proc/mountstats` (per-task)

`fs/proc_namespace.c` provides per-mount-namespace renderings:
- `/proc/mounts` — symlink to `/proc/self/mounts`; legacy mtab format `<dev> <mount-point> <type> <opts> <freq> <passno>`
- `/proc/mountinfo` — extended mount info with mount-id, parent-mount-id, source-fs, root, mount-point, mount-opts, super-opts
- `/proc/mountstats` — per-mount per-file-system statistics

Format byte-identical for each (used by `findmnt`/`mount`/`df`).

### Symlinks: `/proc/{self,thread-self,mounts,net}` (at proc-root)

- `/proc/self` → `<calling-pid>` (cross-ref `proc-task.md`)
- `/proc/thread-self` → `<calling-pid>/task/<calling-tid>` (cross-ref `proc-task.md`)
- `/proc/mounts` → `/proc/self/mounts`
- `/proc/net` → `/proc/self/net` (cross-ref `proc-net.md`)

Identical resolution.

### Refcount + RCU lifecycle

Per-`proc_dir_entry` `refcnt`: incremented on lookup; decremented on file close. `proc_remove` waits for refcnt to drop before freeing.

Per-`proc_inode` lifecycle managed through VFS dentry-cache via standard `iput` path. RCU-side dentry walk for hot-path lookups.

### `proc_create_mount_point`

Special: creates an empty directory in procfs that's intended to be mount-overridden (e.g., `/proc/sys` is a `proc_create_mount_point` so that external sysctlfs can be mounted on top). Identical contract.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `proc_create_data` register under parent->subdir_lock + RCU-readable | `kani::proofs::fs::proc::generic::create_safety` |
| `proc_remove` refcount-drop wait + free | `kani::proofs::fs::proc::generic::remove_safety` |
| Per-pid lookup chain (RCU-side; no use-after-free of struct pid) | `kani::proofs::fs::proc::generic::pid_lookup_safety` |
| `procfs_fill_super` (mount option parsing; bounds-check on hidepid/gid) | `kani::proofs::fs::proc::generic::mount_safety` |
| Per-PDE seq_adapter conversion | `kani::proofs::fs::proc::generic::seq_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-parent subdirectory tree | rb-tree balanced; entries ordered by name | `kani::proofs::fs::proc::generic::subdir_invariants` |
| Per-PDE refcount | refcount > 0 between create and remove; remove waits for drop | `kani::proofs::fs::proc::generic::refcount_invariants` |
| Per-mount option-set | hidepid ∈ {0, 1, 2}; gid valid kgid; subset ∈ {none, pid} | `kani::proofs::fs::proc::generic::mount_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `fs/proc/00-overview.md`)

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-PDE + per-proc_inode + per-pid refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-`proc_dir_entry` + per-`proc_inode` slab caches | § Mandatory |
| **CONSTIFY** | per-PDE `proc_ops` instances `static const` (where possible) | § Mandatory |
| **MEMORY_SANITIZE** | freed PDE state cleared (carries name strings + per-data) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, MEMORY_SANITIZE**: see above
- **USERCOPY**: per-file read/write uses VFS bound-checked accessors
- **SIZE_OVERFLOW**: per-PDE name-length + per-mount option-arithmetic uses checked operators
- **KERNEXEC**: `proc_ops` dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_inode_permission` per-file gate (already standard).
- LSM hook `security_inode_setattr` for chmod/chown on PDE inodes.
- LSM hook `security_sb_mount` for procfs mount.
- Default useful GR-RBAC policy: deny custom `proc_create_data` registrations outside gradm-marked `kernel_admin` role; arbitrary procfs entries are sysadmin-introspection surfaces.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

