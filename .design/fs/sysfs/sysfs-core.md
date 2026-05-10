# Tier-3: fs/sysfs/sysfs-core — sysfs core (dir / file / group / mount / symlink)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/sysfs/dir.c
  - fs/sysfs/file.c
  - fs/sysfs/group.c
  - fs/sysfs/mount.c
  - fs/sysfs/symlink.c
  - fs/sysfs/sysfs.h
  - include/linux/sysfs.h
-->

## Summary
Tier-3 design for the sysfs core layer that builds on top of `kernfs` to provide the kobject-aware directory/file/group/symlink API used by every kernel module exposing `/sys/*` entries:

- **Directory mgmt** (`dir.c`) — `sysfs_create_dir_ns` + `sysfs_remove_dir`
- **File mgmt** (`file.c`) — `sysfs_create_file` + `sysfs_create_files` + `sysfs_remove_file*`
- **Group mgmt** (`group.c`) — `sysfs_create_group` + `sysfs_create_groups` (atomic multi-attribute add)
- **Mount + super_block** (`mount.c`) — `sysfs_init` + `sysfs_get_tree`
- **Symlinks** (`symlink.c`) — `sysfs_create_link` + `sysfs_remove_link`

Plus per-`kobject_type` + per-`attribute_group` registration helpers + the per-attribute show/store callback dispatch.

Sub-tier-3 of `fs/sysfs/00-overview.md`. Pairs with `fs/sysfs/kernfs.md` (underlying VFS layer), `fs/sysfs/kobject.md` (consumer of these APIs).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Directory mgmt | `fs/sysfs/dir.c` |
| File mgmt + per-attribute show/store dispatch | `fs/sysfs/file.c` |
| Attribute group atomic add/remove | `fs/sysfs/group.c` |
| Mount + super_block | `fs/sysfs/mount.c` |
| Symlinks | `fs/sysfs/symlink.c` |
| Internal types | `fs/sysfs/sysfs.h` |
| Public API | `include/linux/sysfs.h` |

## Compatibility contract

### `sysfs_create_dir_ns` / `sysfs_remove_dir`

```c
int sysfs_create_dir_ns(struct kobject *kobj, const void *ns);
void sysfs_remove_dir(struct kobject *kobj);
```

Per-kobject directory creation/removal. NS-tagged for namespace-aware sysfs (e.g., per-netns subdirs under /sys/class/net).

### `struct attribute` (basic per-attribute descriptor)

```c
struct attribute {
    const char *name;
    umode_t mode;
};
```

Per-attribute name + permission mode. Layout-byte-identical.

### `struct kobj_attribute` (kobject-level attribute)

```c
struct kobj_attribute {
    struct attribute attr;
    ssize_t (*show)(struct kobject *kobj, struct kobj_attribute *attr, char *buf);
    ssize_t (*store)(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count);
};
```

Per-attribute show/store callbacks. Buffer is `PAGE_SIZE` (4096); show writes formatted output; store parses input. Identical signatures.

### `struct attribute_group` (named group of attributes)

```c
struct attribute_group {
    const char *name;
    umode_t (*is_visible)(struct kobject *, struct attribute *, int);
    umode_t (*is_bin_visible)(struct kobject *, const struct bin_attribute *, int);
    struct attribute **attrs;
    struct bin_attribute **bin_attrs;
};
```

Group can have:
- `name`: subdirectory name (NULL for inline)
- `attrs`: array of regular attributes
- `bin_attrs`: array of binary attributes (no PAGE_SIZE limit)
- `is_visible`/`is_bin_visible`: optional dynamic-visibility predicate

Atomic add: `sysfs_create_group` adds all attrs; on failure, rolls back partial. Identical algorithm.

### `struct bin_attribute` (binary attribute)

```c
struct bin_attribute {
    struct attribute attr;
    size_t size;
    void *private;
    ssize_t (*read)(struct file *, struct kobject *, struct bin_attribute *, char *, loff_t, size_t);
    ssize_t (*write)(struct file *, struct kobject *, struct bin_attribute *, char *, loff_t, size_t);
    int (*mmap)(struct file *, struct kobject *, struct bin_attribute *, struct vm_area_struct *);
};
```

Used for non-text content (firmware blobs / EEPROM dumps / PCI config space). `read`/`write` accept arbitrary offset+size; `mmap` allows zero-copy access.

### Per-attribute show/store dispatch

```c
static ssize_t sysfs_kf_seq_show(struct seq_file *sf, void *v) {
    struct kobject *kobj = sf->private;
    struct kobj_attribute *kattr = to_kobj_attr(...);
    return kattr->show(kobj, kattr, buf);
}
```

Per-read: kernfs allocates 4KB buffer; calls `show`; returns bytes-written. Per-write: similar but for `store`.

Identical algorithm.

### `sysfs_create_link` / `sysfs_remove_link`

```c
int sysfs_create_link(struct kobject *kobj, struct kobject *target, const char *name);
void sysfs_remove_link(struct kobject *kobj, const char *name);
```

Symlinks linking sysfs entries. Target is computed dynamically from kobjects' relative paths in the tree.

### Mount semantics

`sysfs_init` registers `struct file_system_type sysfs_fs_type` at boot. `mount -t sysfs sysfs /sys`:
- Per-system single mount per-instance (sysfs is "magic": same super_block reused per netns)
- Per-netns mount-time differentiation (per-namespace visibility for /sys/class/net etc.)

### sysfs_chmod_file / sysfs_emit / sysfs_emit_at helpers

```c
int sysfs_chmod_file(struct kobject *kobj, const struct attribute *attr, umode_t mode);
ssize_t sysfs_emit(char *buf, const char *fmt, ...);
ssize_t sysfs_emit_at(char *buf, int at, const char *fmt, ...);
```

`sysfs_emit` is the standard write-into-show-buffer helper that ensures bounded output (≤ PAGE_SIZE - position). Required for safe show callbacks.

### Per-system sysfs ns-info `KERNFS_NS_TAG`

Some sysfs subdirs (notably `/sys/class/net`) are netns-tagged via `kobj->ktype->ns()` operation. Per-netns view sees only own netns's subdirs.

### `to_*_attr` casts (in include/linux/sysfs.h)

Standard casting macros to extract typed attribute from base `struct attribute`:
- `container_of(attr, struct kobj_attribute, attr)` for kobj_attribute
- Similar for `device_attribute`, `driver_attribute`, `bus_attribute`, `class_attribute`

Identical conventions.

## Requirements

- REQ-1: `sysfs_create_dir_ns` / `sysfs_remove_dir` per-kobject directory mgmt; identical signatures.
- REQ-2: `struct attribute` + `struct kobj_attribute` + `struct attribute_group` + `struct bin_attribute` byte-identical layout.
- REQ-3: Per-attribute show/store callback dispatch; PAGE_SIZE buffer; identical algorithm.
- REQ-4: `sysfs_create_group` atomic multi-attribute add; on failure rolls back partial.
- REQ-5: `is_visible` / `is_bin_visible` dynamic-visibility predicates.
- REQ-6: bin_attribute read/write/mmap callbacks (non-PAGE_SIZE-bounded I/O).
- REQ-7: `sysfs_create_link` / `sysfs_remove_link` symlink management.
- REQ-8: Mount semantics: `sysfs_init` register; `sysfs_get_tree` netns-aware mount.
- REQ-9: `sysfs_emit` + `sysfs_emit_at` bounded-output helpers.
- REQ-10: `KERNFS_NS_TAG` per-netns view; per-namespace subdirs only visible from own netns.
- REQ-11: Per-system single-mount semantics (sysfs is "magic").
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct attribute` + `kobj_attribute` + `attribute_group` + `bin_attribute` byte-identical. (covers REQ-2)
- [ ] AC-2: `sysfs_create_dir_ns` test: synthetic module creates `/sys/test_dir`; subsequent `ls /sys/test_dir` confirms. (covers REQ-1)
- [ ] AC-3: kobj_attribute test: register attribute with show callback returning "hello\n"; `cat /sys/.../attr` returns "hello\n". (covers REQ-3)
- [ ] AC-4: store test: register store callback that updates module variable; `echo 42 > /sys/.../attr`; subsequent `cat` returns "42\n". (covers REQ-3)
- [ ] AC-5: attribute_group atomic test: register group of 3 attributes where 2nd's create fails; cleanup rolls back 1st; final state has 0 attributes. (covers REQ-4)
- [ ] AC-6: is_visible test: register group with `is_visible` returning 0 for some attrs based on per-kobj state; those attrs not visible in `ls`. (covers REQ-5)
- [ ] AC-7: bin_attribute test: register binary attribute with read+write+mmap callbacks; `dd if=/sys/.../bin_attr` works; mmap returns valid mapping. (covers REQ-6)
- [ ] AC-8: symlink test: `sysfs_create_link(parent, target, "link_name")`; `readlink /sys/.../link_name` returns target's path. (covers REQ-7)
- [ ] AC-9: per-netns test: `/sys/class/net` shows only current-netns ifaces; netns A's ifaces not visible from netns B. (covers REQ-10)
- [ ] AC-10: sysfs_emit test: show callback uses sysfs_emit; output ≤ PAGE_SIZE always. (covers REQ-9)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::fs::sysfs::dir::CreateDir` — `sysfs_create_dir_ns`
- `kernel::fs::sysfs::dir::RemoveDir` — `sysfs_remove_dir`
- `kernel::fs::sysfs::file::CreateFile` — `sysfs_create_file*`
- `kernel::fs::sysfs::file::ShowStore` — per-attribute show/store dispatch
- `kernel::fs::sysfs::group::CreateGroup` — `sysfs_create_group(s)`
- `kernel::fs::sysfs::group::Visibility` — `is_visible` / `is_bin_visible` evaluation
- `kernel::fs::sysfs::file::BinAttribute` — bin_attribute read/write/mmap
- `kernel::fs::sysfs::symlink::CreateLink` — `sysfs_create_link`
- `kernel::fs::sysfs::mount::SysfsMount` — `sysfs_init` + `sysfs_get_tree`
- `kernel::fs::sysfs::file::Emit` — `sysfs_emit` / `sysfs_emit_at`
- `kernel::fs::sysfs::ns::NsTag` — `KERNFS_NS_TAG` per-netns

### Locking and concurrency

- **Per-kernfs_node `kn->iattr->ia_lock`** (mutex): per-attribute mutator
- **Per-parent kernfs_node tree-lock**: directory mutation (cross-ref `kernfs.md`)
- **RCU**: hot-path lookup RCU-side
- **Active-path locking** (cross-ref `kernfs.md`): concurrent show/store vs. removal

### Error handling

- `Err(EEXIST)` — duplicate name in parent
- `Err(ENOENT)` — entry not found
- `Err(EACCES)` — permission denied
- `Err(EINVAL)` — bad attribute / bad mode
- `Err(ENOMEM)` — alloc fail
- `Err(EROFS)` — write to read-only attribute

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `sysfs_create_dir_ns` register under parent tree-lock | `kani::proofs::fs::sysfs::core::create_dir_safety` |
| `sysfs_create_group` atomic add (rollback on partial fail) | `kani::proofs::fs::sysfs::core::group_safety` |
| Per-attribute show/store buffer-bounds (PAGE_SIZE clamp) | `kani::proofs::fs::sysfs::core::show_store_safety` |
| bin_attribute read/write offset+size bounds | `kani::proofs::fs::sysfs::core::bin_attr_safety` |
| Symlink create + target resolution | `kani::proofs::fs::sysfs::core::symlink_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/fs/kernfs_active_path.tla` from `fs/sysfs/kernfs.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-parent attribute list | every attribute has unique name within parent | `kani::proofs::fs::sysfs::core::attr_invariants` |
| attribute_group | every attr has matching is_visible result; rollback restores tree | `kani::proofs::fs::sysfs::core::group_invariants` |
| bin_attribute size | post-write offset ≤ size; mmap range ⊆ [0, size) | `kani::proofs::fs::sysfs::core::bin_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `fs/sysfs/00-overview.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-attribute_group + per-bin_attribute + per-kobj_attribute instances `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed seq_file scratch buffers cleared (carry potentially-sensitive attribute values) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, CONSTIFY**: see above + per-kernfs_node slab
- **USERCOPY**: per-attribute show/store uses bound-checked accessors via sysfs_emit
- **SIZE_OVERFLOW**: bin_attribute offset+size arithmetic uses checked operators
- **KERNEXEC**: per-attribute dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_inode_permission` per-file gate.
- LSM hook `security_inode_setattr` for chmod on attribute files.
- Default useful GR-RBAC policy: deny per-attribute writes outside gradm-marked roles for sensitive paths (e.g., `/sys/power/state`, `/sys/kernel/livepatch/*`, `/sys/kernel/debug/*`).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — sysfs core API exhaustively specified by upstream + driver-developer test corpus)

## Out of Scope

- kernfs underlying (cross-ref `fs/sysfs/kernfs.md`)
- kobject framework (cross-ref `fs/sysfs/kobject.md`)
- Per-bus/class/device-specific sysfs entries (cross-ref individual subsystem Tier-3s)
- 32-bit-only paths
- Implementation code
