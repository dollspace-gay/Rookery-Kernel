# Tier-3: fs/sysfs/kernfs — kernfs underlying VFS layer (sysfs/cgroupfs/configfs base)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/kernfs/dir.c
  - fs/kernfs/file.c
  - fs/kernfs/inode.c
  - fs/kernfs/mount.c
  - fs/kernfs/symlink.c
  - fs/kernfs/kernfs-internal.h
  - include/linux/kernfs.h
-->

## Summary
Tier-3 design for kernfs — the underlying VFS layer extracted from sysfs in v3.14 to provide a reusable in-memory pseudo-filesystem framework. Used by:
- **sysfs** (cross-ref `fs/sysfs/sysfs-core.md`)
- **cgroupfs** (cgroup-v2 — cross-ref future `kernel/cgroup/00-overview.md`)
- **configfs** (cross-ref future `fs/configfs/00-overview.md`)
- **tracefs** (cross-ref future `fs/tracefs/00-overview.md`)

Provides:
- `struct kernfs_node` — per-node tree element
- `struct kernfs_root` — per-fs-instance root
- Tree management (insert/remove/lookup/iterate)
- Active-path locking (concurrent reader vs. mutator coexistence — the cornerstone of safe attribute-file removal during in-progress reads)
- Per-node `kernfs_open_file` / `kernfs_open_node` per-fd state
- Super_block + inode allocation
- VFS dispatch via `kernfs_file_ops` / `kernfs_dir_ops` / `kernfs_iops`

Owns the mandatory `models/fs/kernfs_active_path.tla` TLA+ model.

Sub-tier-3 of `fs/sysfs/00-overview.md`. Pairs with `fs/sysfs/sysfs-core.md` (consumer), `fs/sysfs/kobject.md` (also a consumer).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Directory mgmt + tree insert/remove + lookup | `fs/kernfs/dir.c` |
| Per-attribute file ops + active-path locking | `fs/kernfs/file.c` |
| Inode lifecycle + permission ops | `fs/kernfs/inode.c` |
| Mount + super_block | `fs/kernfs/mount.c` |
| Symlink ops | `fs/kernfs/symlink.c` |
| Internal types | `fs/kernfs/kernfs-internal.h` |
| Public API | `include/linux/kernfs.h` |

## Compatibility contract

### `struct kernfs_node`

```c
struct kernfs_node {
    atomic_t count;          /* refcount */
    atomic_t active;         /* active-path users */
    struct kernfs_node *parent;
    const char *name;
    struct rb_node rb;
    const void *ns;
    unsigned int hash;
    unsigned short flags;
    umode_t mode;
    union {
        struct kernfs_elem_dir dir;
        struct kernfs_elem_symlink symlink;
        struct kernfs_elem_attr attr;
    };
    void *priv;
    u64 id;
    struct kernfs_iattrs *iattr;
    struct rcu_head rcu;
};
```

Layout-equivalent for first cache-line. Nodes can be:
- Directory (`KERNFS_DIR`): contains rb-tree of children
- File (`KERNFS_FILE`): backed by `struct kernfs_ops` callbacks
- Symlink (`KERNFS_LINK`): points to target node

### Active-path locking — the cornerstone

When a userspace task opens a kernfs file and is mid-read, kernel code calling `kernfs_remove(node)` must wait until the read completes (otherwise use-after-free). Active-path locking implements this:

- Per-`kernfs_node` `active` atomic counter
- `kernfs_get_active(node)` — atomic-inc with check-deactivated
- `kernfs_put_active(node)` — atomic-dec
- `kernfs_drain(node)` — atomically marks deactivated + waits for active=0

Concurrent semantics: a reader holds `active` while in show/store callback; a remover sets `KERNFS_REMOVED` flag (preventing new active-acquires) + waits for active to drain.

Identical algorithm — provably safe under concurrent access.

### `struct kernfs_ops` per-attribute callbacks

```c
struct kernfs_ops {
    int (*open)(struct kernfs_open_file *of);
    void (*release)(struct kernfs_open_file *of);
    int (*seq_show)(struct seq_file *sf, void *v);
    void *(*seq_start)(struct seq_file *sf, loff_t *ppos);
    void *(*seq_next)(struct seq_file *sf, void *v, loff_t *ppos);
    void (*seq_stop)(struct seq_file *sf, void *v);
    ssize_t (*read)(struct kernfs_open_file *of, char *buf, size_t bytes, loff_t off);
    ssize_t (*write)(struct kernfs_open_file *of, char *buf, size_t bytes, loff_t off);
    __poll_t (*poll)(struct kernfs_open_file *of, struct poll_table_struct *pt);
    int (*mmap)(struct kernfs_open_file *of, struct vm_area_struct *vma);
    loff_t (*llseek)(struct kernfs_open_file *of, loff_t offset, int whence);
    bool prealloc;
    size_t atomic_write_len;
    /* etc. */
};
```

Layout-equivalent. Per-file-type kernfs callbacks dispatch through this vtable.

### `kernfs_create_dir_ns` / `kernfs_create_file_ns` / `kernfs_create_link`

```c
struct kernfs_node *kernfs_create_dir_ns(struct kernfs_node *parent, const char *name, umode_t mode,
                                          kuid_t uid, kgid_t gid, void *priv, const void *ns);
struct kernfs_node *kernfs_create_file_ns(struct kernfs_node *parent, const char *name, umode_t mode,
                                           kuid_t uid, kgid_t gid, loff_t size,
                                           const struct kernfs_ops *ops, void *priv,
                                           const void *ns, struct lock_class_key *key);
struct kernfs_node *kernfs_create_link(struct kernfs_node *parent, const char *name, struct kernfs_node *target);
```

Identical signatures.

### `kernfs_remove` + `kernfs_remove_by_name_ns`

```c
void kernfs_remove(struct kernfs_node *kn);
int kernfs_remove_by_name_ns(struct kernfs_node *parent, const char *name, const void *ns);
```

Removal: marks node KERNFS_REMOVED; drains active; recursively removes children for directories. Wait-for-active is the locking-correctness invariant.

### `struct kernfs_root` per-fs-instance

```c
struct kernfs_root {
    struct kernfs_node *kn;
    unsigned int flags;
    struct idr ino_idr;
    spinlock_t kernfs_idr_lock;
    u32 last_id_lowbits;
    u32 id_highbits;
    struct kernfs_syscall_ops *syscall_ops;
    struct list_head supers;
    wait_queue_head_t deactivate_waitq;
    struct rw_semaphore kernfs_rwsem;
    struct rw_semaphore kernfs_iattr_rwsem;
    struct rw_semaphore kernfs_supers_rwsem;
};
```

Per-fs-instance state. Per-kernfs-instance flags include `KERNFS_ROOT_CREATE_DEACTIVATED` (sysfs-style: created-but-not-yet-active until explicit activate) and `KERNFS_ROOT_EXTRA_OPEN_PERM_CHECK`.

### Per-node namespace (`KERNFS_NS_TAG`)

When a parent has `KERNFS_NS_TAG` flag, child lookup matches by `(name, ns_tag)` tuple instead of just `name`. Used by sysfs for per-netns differentiation of `/sys/class/net/<iface>` etc.

### Per-`kernfs_open_file` per-fd state

```c
struct kernfs_open_file {
    struct kernfs_node *kn;
    struct file *file;
    struct seq_file *seq_file;
    void *priv;
    char *prealloc_buf;
    size_t atomic_write_len;
    bool mmapped;
    bool released;
    const struct vm_operations_struct *vm_ops;
    struct mutex mutex;
    struct mutex prealloc_mutex;
    int event;
    struct rcu_head rcu_head;
};
```

Per-fd state. Layout-equivalent.

### Per-system mount: kernfs is per-instance

Each consumer (sysfs / cgroupfs / configfs / tracefs) has its own `kernfs_root` instance. Per-system mount through `kernfs_get_tree`.

## Requirements

- REQ-1: `struct kernfs_node` + `struct kernfs_root` + `struct kernfs_open_file` + `struct kernfs_ops` first-cache-line layout-equivalent.
- REQ-2: Active-path locking: per-node `active` counter; `kernfs_get_active` / `_put_active` / `_drain` semantics identical.
- REQ-3: `kernfs_create_dir_ns` / `kernfs_create_file_ns` / `kernfs_create_link` registration; identical signatures.
- REQ-4: `kernfs_remove` + `kernfs_remove_by_name_ns`: drain active before free; recursive for directories.
- REQ-5: Per-node namespace via `KERNFS_NS_TAG`: child lookup matches by `(name, ns_tag)` tuple.
- REQ-6: Per-`kernfs_open_file` per-fd state: per-mutator path serialization via `mutex`; preallocated buffer for `atomic_write_len`.
- REQ-7: Mount via `kernfs_get_tree`: per-instance super_block; per-instance kernfs_root.
- REQ-8: Inode lifecycle: `kernfs_iattrs` lazy-allocated for files with custom permissions; per-fs lookup via `kernfs_idr`.
- REQ-9: Symlink resolution: `kernfs_symlink` resolves target via parent + relative path traversal.
- REQ-10: TLA+ model `models/fs/kernfs_active_path.tla` (mandatory per `fs/sysfs/00-overview.md` Layer 2) — proves: concurrent reader vs. mutator coexistence is sound; no use-after-free during in-progress read on file removal.
- REQ-11: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct kernfs_node` + `kernfs_root` + `kernfs_open_file` + `kernfs_ops` byte-identical first cache-line. (covers REQ-1)
- [ ] AC-2: Active-path test: reader thread holds `kernfs_get_active`; concurrent `kernfs_remove` blocks until reader puts active; release allows remove to proceed. (covers REQ-2, REQ-4)
- [ ] AC-3: Deep removal test: `kernfs_remove(parent)` recursively removes 1000 children atomically. (covers REQ-4)
- [ ] AC-4: NS_TAG test: parent with KERNFS_NS_TAG; same-name children with different ns_tags coexist; lookup matches correct tag. (covers REQ-5)
- [ ] AC-5: Atomic write test: `atomic_write_len=4096`; userspace `write(fd, buf, 1024)` succeeds; `write(fd, buf, 5000)` returns -EINVAL or partial. (covers REQ-6)
- [ ] AC-6: Mount test: synthetic test consumer registers kernfs_root; `mount -t consumerfs ...` works; subsequent operations on the mount succeed. (covers REQ-7)
- [ ] AC-7: Per-fd state test: 2 concurrent readers of same kernfs file; each has independent `kernfs_open_file` (no shared state). (covers REQ-6)
- [ ] AC-8: Symlink test: `kernfs_create_link(parent, "link", target)`; readlink/lookup via "link" returns target's path. (covers REQ-9)
- [ ] AC-9: Concurrent stress: 1000 threads create+remove attribute files concurrently with 100 readers; no use-after-free, no leaks. (covers REQ-2, REQ-4)
- [ ] AC-10: TLA+ `models/fs/kernfs_active_path.tla` proves: at every reachable state, in-progress readers' nodes are not in REMOVED state with active=0; remove waits for drain. (covers REQ-10)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

## Architecture

### Rust module organization

- `kernel::fs::kernfs::Node` — `struct kernfs_node`
- `kernel::fs::kernfs::Root` — `struct kernfs_root`
- `kernel::fs::kernfs::OpenFile` — `struct kernfs_open_file`
- `kernel::fs::kernfs::Ops` — `struct kernfs_ops` vtable
- `kernel::fs::kernfs::Active` — active-path counter + drain
- `kernel::fs::kernfs::Tree` — per-parent rb-tree of children
- `kernel::fs::kernfs::Create` — create_dir / create_file / create_link
- `kernel::fs::kernfs::Remove` — remove + drain + recursive
- `kernel::fs::kernfs::NsTag` — KERNFS_NS_TAG lookup
- `kernel::fs::kernfs::Inode` — inode lifecycle + iattrs
- `kernel::fs::kernfs::Mount` — super_block + get_tree
- `kernel::fs::kernfs::Symlink` — symlink resolution

### Locking and concurrency

- **Per-`kernfs_root` `kernfs_rwsem`** (rwsem): tree mutator
- **Per-`kernfs_node` `active`** (atomic): active-path counter
- **Per-`kernfs_node` `iattr` rwsem**: per-iattr mutator (lazy-alloc)
- **Per-`kernfs_open_file` `mutex`**: per-fd serialization (write/seek)
- **`deactivate_waitq`**: wait-for-active-drain
- **RCU**: tree lookup hot path RCU-side

### Error handling

- `Err(EEXIST)` — duplicate name in parent
- `Err(ENOENT)` — node not found
- `Err(EBUSY)` — active-path drain timeout (rarely)
- `Err(EACCES)` — permission denied
- `Err(EINVAL)` — bad node-type / bad ops
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Active-path counter (atomic-inc + check-deactivated; no race window) | `kani::proofs::fs::kernfs::active_safety` |
| Tree insert/remove under kernfs_rwsem | `kani::proofs::fs::kernfs::tree_safety` |
| Per-fd open-file state mutator under per-fd mutex | `kani::proofs::fs::kernfs::open_file_safety` |
| Symlink-target resolution (no infinite loop on cyclic links) | `kani::proofs::fs::kernfs::symlink_safety` |
| Recursive remove (depth-bounded; no use-after-free of children) | `kani::proofs::fs::kernfs::remove_safety` |

### Layer 2: TLA+ models

- `models/fs/kernfs_active_path.tla` (mandatory per `fs/sysfs/00-overview.md` Layer 2) — proves active-path-locking semantics. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-parent rb-tree | balanced; entries ordered by hash + name; no duplicate names within parent (modulo NS_TAG) | `kani::proofs::fs::kernfs::tree_invariants` |
| Per-node refcount + active | refcount ≥ 1 between create and final remove; active ≥ 0; remove blocks while active > 0 | `kani::proofs::fs::kernfs::ref_invariants` |
| Per-node KERNFS_REMOVED flag | once set, never unset; new active-acquires fail | `kani::proofs::fs::kernfs::removed_invariants` |

### Layer 4: Functional correctness (declared in `fs/sysfs/00-overview.md` Layer 4)

- **Active-path locking soundness theorem** via TLA+ refinement — proves: implementation refines the abstract active-path-locking model; no UAF.
- **kobject refcount soundness theorem** (cross-ref `fs/sysfs/00-overview.md`) — consumed by kobject.md.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-node refcount + active counter use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-`kernfs_node` + per-`kernfs_open_file` + per-`kernfs_iattrs` slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed nodes cleared (carry name + per-node priv data) | § Default-on configurable off |
| **CONSTIFY** | per-`kernfs_ops` instances `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, CONSTIFY**: see above
- **USERCOPY**: per-file read/write uses bound-checked accessors
- **SIZE_OVERFLOW**: per-node id + per-tree depth arithmetic uses checked operators
- **KERNEXEC**: per-ops dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook `security_inode_permission` — per-file gate.
- Default GR-RBAC policy: empty (consumer-side policies via sysfs/cgroupfs).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

kernfs is the underlying VFS-backed in-memory tree for sysfs, cgroupfs, and configfs; grsec hardened the node lifecycle (active-counter draining, ops-vtable immutability). Rookery commits:

- **PAX_USERCOPY** — per-`kernfs_open_file` read/write buffer sized at `seq_file` allocation; copy_to_user/copy_from_user bounded by that size.
- **PAX_KERNEXEC** — every `kernfs_ops` instance is `static const`; per-node `priv` may be mutable but the vtable is rodata.
- **PAX_RANDKSTACK** — kernfs open/read/write paths run under randomized kstack offset.
- **PAX_REFCOUNT** — `kernfs_node.count` and `kernfs_node.active` are saturating refcount; `kernfs_active_drain` is a barrier against use-after-free during `kernfs_remove`.
- **PAX_MEMORY_SANITIZE** — freed `kernfs_node` slabs zeroed; per-node `name` and `priv` pointer cleared before slab reuse so prior-node identity cannot bleed.
- **PAX_UDEREF** — read/write callbacks receive userspace pointer as `void __user *` and consume via `copy_*_user`; raw deref is a build break.
- **PAX_RAP/kCFI** — `kernfs_ops.{open,release,seq_show,read,write,poll,mmap}` are all CFI-typed and pinned `__ro_after_init`.
- **GRKERNSEC_HIDESYM** — kernfs read paths never emit a kernel pointer unless the per-file `seq_show` explicitly opted in and the reader has CAP_SYSLOG.
- **GRKERNSEC_DMESG** — node-remove races and active-drain timeouts never log the node path to dmesg.
- **kernfs node refcount audit** — `kernfs_get`/`kernfs_put` use saturating refcount; `kernfs_active_drain` completion handshake guarantees no `kernfs_ops.read`/`write` can race a `kernfs_remove`; per-node `active` counter is the explicit barrier grsec required.
- **sysfs_ops PAX_RAP** — the `sysfs_ops` shim (`sysfs_kf_seq_show`, `sysfs_kf_write`, etc.) that bridges kernfs to per-kobject attribute show/store is CFI-typed; per-attribute `show`/`store` function pointers re-checked against CFI allow-list at every kernfs `seq_show` dispatch.
- **kernfs_create_link symlink target validation** — symlink targets resolved within the same kernfs root; cross-root or out-of-tree targets rejected to prevent path-confusion attacks.
- **Per-`kernfs_root` lockdown** — root pointer stored in `__ro_after_init` for sysfs/cgroupfs/configfs; module-provided kernfs trees must register a CFI-typed root.

Rationale: kernfs is the shared lifecycle engine for three high-attack-surface filesystems (sysfs, cgroupfs, configfs); a single corrupted ops-vtable here yields arbitrary call from any of those filesystems. The grsec-equivalent contract pins ops-vtables to rodata and audits the active-counter draining at the kernfs layer once, rather than re-implementing the gate three times.

## Open Questions

(none — kernfs framework exhaustively specified by upstream + sysfs/cgroup-v2/configfs interop)

## Out of Scope

- sysfs core (cross-ref `fs/sysfs/sysfs-core.md`)
- cgroupfs (cross-ref future `kernel/cgroup/00-overview.md`)
- configfs / tracefs (cross-ref future Tier-3s)
- 32-bit-only paths
- Implementation code
