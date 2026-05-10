---
title: "Tier-3: fs/sysfs/kobject — kobject framework + uevent emitter"
tags: ["design-doc", "tier-3", "fs", "sysfs"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-3 design for the kobject framework + uevent infrastructure. The kobject framework is the foundation of Linux's device-driver model: every device/driver/bus/class is represented by a `struct kobject` (or a struct embedding kobject). kobjects:
- Have refcounts (kref-managed)
- Have parents (forming the device tree)
- Have ktype (describing attributes + show/store ops)
- Belong to a kset (per-bus / per-class)
- Emit uevents on lifecycle events (add / remove / change / move / online / offline / bind / unbind)

The uevent infrastructure (`kobject_uevent.c`) emits these events to userspace consumers (udev, systemd-udevd) via NETLINK_KOBJECT_UEVENT broadcast — every plug-and-play device-handling rule depends on this format.

Sub-tier-3 of `fs/sysfs/00-overview.md`. Pairs with `fs/sysfs/sysfs-core.md` (consumer of kobjects via sysfs registration), `fs/sysfs/kernfs.md` (lower-layer storage).

### Requirements

- REQ-1: `struct kobject` + `struct kobj_type` + `struct kset` + `struct sysfs_ops` + `struct kobj_uevent_env` byte-identical first-cache-line layout.
- REQ-2: Lifecycle: `kobject_init` → `kobject_add` (with sysfs dir create + add uevent emit) → `kobject_del` (sysfs dir remove + remove uevent emit) → `kobject_put` (kref decrement + release on zero).
- REQ-3: `kref` semantics: kref_init / get / put / get_unless_zero per upstream signatures.
- REQ-4: 8 uevent action constants (ADD/REMOVE/CHANGE/MOVE/ONLINE/OFFLINE/BIND/UNBIND); identical strings.
- REQ-5: `kobject_uevent` + `kobject_uevent_env` build event message + broadcast via NETLINK_KOBJECT_UEVENT + write to `/sys/.../uevent` file.
- REQ-6: NETLINK_KOBJECT_UEVENT message wire format byte-identical to upstream (text key=value NUL-separated, ACTION/DEVPATH/SUBSYSTEM/SEQNUM mandatory).
- REQ-7: `struct kset_uevent_ops` per-kset hooks (filter / name / uevent) byte-identical signatures.
- REQ-8: UEVENT_BUFFER_SIZE = 2048 cap on event total size.
- REQ-9: Per-attribute show/store via `sysfs_ops` vtable; per-ktype dispatch identical.
- REQ-10: `default_groups` per-ktype auto-add identical.
- REQ-11: `kobject_ns` namespace-aware tagging via `ktype->namespace` callback.
- REQ-12: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct kobject` + `kobj_type` + `kset` + `sysfs_ops` + `kobj_uevent_env` byte-identical. (covers REQ-1)
- [ ] AC-2: Lifecycle test: synthetic module: `kobject_init + kobject_add` → `/sys/test_kobj/` exists + udev sees "add" uevent; `kobject_del` → uevent "remove" emitted + dir gone; `kobject_put` → release called on refcount=0. (covers REQ-2, REQ-4, REQ-5)
- [ ] AC-3: kref test: `kobject_get` 5 times + `kobject_put` 5 times → refcount drops to 0 → release called once. (covers REQ-3)
- [ ] AC-4: Uevent format test: capture NETLINK_KOBJECT_UEVENT message; format byte-identical (ACTION + DEVPATH + SUBSYSTEM + SEQNUM headers). (covers REQ-5, REQ-6)
- [ ] AC-5: Filter test: synthetic kset with `filter` returning 0 for some kobjects; those kobjects don't emit uevents. (covers REQ-7)
- [ ] AC-6: UEVENT_BUFFER_SIZE test: synthetic uevent with > 2048 bytes envp → -ENOMEM or truncated; per upstream's behavior. (covers REQ-8)
- [ ] AC-7: show/store test: kobj_attribute with show callback → cat returns formatted output; store callback → echo writes value. (covers REQ-9)
- [ ] AC-8: default_groups test: define ktype with default_groups; subsequent kobject_add auto-creates group files. (covers REQ-10)
- [ ] AC-9: namespace-aware test: per-netns kobject; `kobject_ns(kobj)` returns netns pointer; sysfs subdirs only visible from owning netns. (covers REQ-11)
- [ ] AC-10: udev integration test: real udev daemon receives uevents; udev rules match per-attribute and trigger expected actions. (covers REQ-5, REQ-6)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-12)

### Architecture

### Rust module organization

- `kernel::lib::kobject::Kobject` — `struct kobject`
- `kernel::lib::kobject::KobjType` — `struct kobj_type`
- `kernel::lib::kobject::Kset` — `struct kset`
- `kernel::lib::kobject::Lifecycle` — init / add / del / put
- `kernel::lib::kobject::Kref` — kref_init / get / put / get_unless_zero
- `kernel::lib::kobject::SysfsOps` — `struct sysfs_ops` per-ktype dispatch
- `kernel::lib::kobject::DefaultGroups` — auto-create per-ktype default groups
- `kernel::lib::kobject::Namespace` — kobject_ns tagging
- `kernel::lib::kobject::uevent::Action` — KOBJ_ADD/REMOVE/CHANGE/MOVE/ONLINE/OFFLINE/BIND/UNBIND
- `kernel::lib::kobject::uevent::Env` — `struct kobj_uevent_env` builder
- `kernel::lib::kobject::uevent::Filter` — per-kset filter hook
- `kernel::lib::kobject::uevent::Broadcast` — NETLINK_KOBJECT_UEVENT emit

### Locking and concurrency

- **Per-kset `list_lock`** (spinlock): kset member list mutator
- **`uevent_sock_mutex`** (mutex): per-netns uevent socket list
- **`kobject_actions_lock`** (mutex): action-string lookup
- **kref**: lockless atomic refcount

### Error handling

- `Err(EEXIST)` — duplicate name in parent kset
- `Err(ENOMEM)` — alloc fail
- `Err(ENODEV)` — kobject not initialized
- `Err(EINVAL)` — bad action / bad name

### Out of Scope

- sysfs core (cross-ref `fs/sysfs/sysfs-core.md`)
- kernfs underlying (cross-ref `fs/sysfs/kernfs.md`)
- Per-driver implementations (cross-ref individual driver Tier-3s when added)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| kobject framework: kobj_init / _add / _del + ktype + kset | `lib/kobject.c` |
| uevent emitter: kobject_uevent + NETLINK_KOBJECT_UEVENT | `lib/kobject_uevent.c` |
| Public API | `include/linux/kobject.h` |

### compatibility contract

### `struct kobject`

```c
struct kobject {
    const char *name;
    struct list_head entry;
    struct kobject *parent;
    struct kset *kset;
    const struct kobj_type *ktype;
    struct kernfs_node *sd;          /* sysfs directory entry */
    struct kref kref;
    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;
};
```

Layout-byte-identical for first cache-line.

### `struct kobj_type` (ktype)

```c
struct kobj_type {
    void (*release)(struct kobject *kobj);
    const struct sysfs_ops *sysfs_ops;
    const struct attribute_group **default_groups;
    const struct kobj_ns_type_operations *(*child_ns_type)(const struct kobject *kobj);
    const void *(*namespace)(const struct kobject *kobj);
    void (*get_ownership)(const struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```

Per-kobject-type vtable. Released kobject calls `release` (often `kfree(container_of(kobj, ...))`).

Layout-equivalent.

### `struct kset` (set/container of kobjects)

```c
struct kset {
    struct list_head list;
    spinlock_t list_lock;
    struct kobject kobj;
    const struct kset_uevent_ops *uevent_ops;
};
```

Per-bus / per-class container; kobjects added to a kset appear in the corresponding sysfs directory.

### Lifecycle: kobject_init → kobject_add → kobject_del → kobject_put

```c
void kobject_init(struct kobject *kobj, const struct kobj_type *ktype);
int kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...);
void kobject_del(struct kobject *kobj);
void kobject_put(struct kobject *kobj);
```

After `kobject_init`, kobject is in `INITIALIZED` state. `kobject_add` registers + creates sysfs dir + emits `add` uevent. `kobject_del` un-registers + emits `remove` uevent. `kobject_put` decrements refcount; on reach 0, calls `ktype->release`.

Identical lifecycle.

### `kref` reference count

```c
void kref_init(struct kref *kref);
void kref_get(struct kref *kref);
int kref_put(struct kref *kref, void (*release)(struct kref *kref));
int kref_get_unless_zero(struct kref *kref);
```

Embedded in kobject. kref-based refcount. Identical semantics.

### Uevent events

| Event | Constant | When |
|---|---|---|
| `KOBJ_ADD` | "add" | Kobject added to sysfs |
| `KOBJ_REMOVE` | "remove" | Kobject removed |
| `KOBJ_CHANGE` | "change" | Kobject state changed (e.g., link up/down) |
| `KOBJ_MOVE` | "move" | Kobject moved within tree |
| `KOBJ_ONLINE` | "online" | Device brought online |
| `KOBJ_OFFLINE` | "offline" | Device taken offline |
| `KOBJ_BIND` | "bind" | Driver bound to device |
| `KOBJ_UNBIND` | "unbind" | Driver unbound |

Identical event set.

### `kobject_uevent` + `kobject_uevent_env`

```c
int kobject_uevent(struct kobject *kobj, enum kobject_action action);
int kobject_uevent_env(struct kobject *kobj, enum kobject_action action, char *envp[]);
```

Builds uevent message + broadcasts via NETLINK_KOBJECT_UEVENT to subscribed userspace listeners (and writes to `/sys/.../uevent` file).

Format: text key=value pairs separated by NULs:
```
ACTION=add\0
DEVPATH=/devices/platform/...\0
SUBSYSTEM=net\0
[per-attr KEY=VALUE...\0]
SEQNUM=12345\0
```

Wire format byte-identical so udev rules work.

### NETLINK_KOBJECT_UEVENT family

Per-netns uevent broadcast: per-netns netlink socket; message format byte-identical. udev binds to receive uevents matching its rules.

### `struct kset_uevent_ops`

```c
struct kset_uevent_ops {
    int (*const filter)(const struct kobject *kobj);
    const char *(*const name)(const struct kobject *kobj);
    int (*const uevent)(const struct kobject *kobj, struct kobj_uevent_env *env);
};
```

Per-kset hook to:
- `filter` — decide if event is emitted at all
- `name` — override default subsystem name
- `uevent` — add per-kset extra envp variables

### `kobj_uevent_env` builder

```c
struct kobj_uevent_env {
    char *argv[3];
    char *envp[UEVENT_NUM_ENVP];
    int envp_idx;
    char buf[UEVENT_BUFFER_SIZE];
    int buflen;
};
```

Per-emit scratch buffer. UEVENT_BUFFER_SIZE = 2048 (caps total event size).

### Per-attribute show/store via `sysfs_ops`

```c
struct sysfs_ops {
    ssize_t (*show)(struct kobject *kobj, struct attribute *attr, char *buf);
    ssize_t (*store)(struct kobject *kobj, struct attribute *attr, const char *buf, size_t count);
};
```

Per-ktype dispatch: when read/write on a sysfs attribute file, kernel calls `kobj->ktype->sysfs_ops->show/store(kobj, attr, buf)`. The per-ktype implementation typically casts `attr` to its specific type (e.g., `kobj_attribute`) and dispatches to per-attribute callback.

### Default attribute groups

`ktype->default_groups[]` — per-ktype attributes auto-created when the kobject is added. Common pattern: per-bus / per-class declares default_groups; every device of that type gets these files automatically.

### `kobject_ns` namespace tagging

For namespace-aware kobjects (e.g., per-netns net-interfaces), `kobject->ktype->namespace(kobj)` returns the namespace pointer. Used by sysfs to tag kernfs_nodes with `KERNFS_NS_TAG`.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| kobject_add/del lifecycle (refcount sound across add/del/put) | `kani::proofs::lib::kobject::lifecycle_safety` |
| kref atomic-acquire/release (no overflow on get; no underflow on put) | `kani::proofs::lib::kobject::kref_safety` |
| Uevent envp builder (UEVENT_BUFFER_SIZE bound) | `kani::proofs::lib::kobject::env_safety` |
| Per-kset filter dispatch + name override | `kani::proofs::lib::kobject::filter_safety` |
| NETLINK_KOBJECT_UEVENT broadcast (per-netns isolation) | `kani::proofs::lib::kobject::broadcast_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/fs/kernfs_active_path.tla` from `kernfs.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-kobject state | flags consistent (state_initialized → in_sysfs ⊕ post_remove); add_uevent_sent ↔ remove_uevent_sent | `kani::proofs::lib::kobject::state_invariants` |
| Per-kset member list | every member's `kset` pointer points back to this kset | `kani::proofs::lib::kobject::member_invariants` |
| Per-uevent envp | envp_idx ≤ UEVENT_NUM_ENVP; total bytes ≤ UEVENT_BUFFER_SIZE | `kani::proofs::lib::kobject::env_invariants` |

### Layer 4: Functional correctness (declared in `fs/sysfs/00-overview.md` Layer 4)

- **kobject refcount soundness theorem** via Verus — proves: per-kobject refcount transitions UNINIT → INITED → REFCOUNT_OK → REFCOUNT_DEAD; freed only after refcount=0 + release called. Owned here.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-kobject kref + per-kset kobj refcount use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-kobj_uevent_env scratch slab cache | § Mandatory |
| **CONSTIFY** | per-ktype + per-kset_uevent_ops + per-default_groups instances `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed kobj_uevent_env + per-kobject name string cleared (carry sysfs paths + per-attr data) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, MEMORY_SANITIZE**: see above
- **USERCOPY**: NETLINK_KOBJECT_UEVENT message build uses bound-checked accessors
- **SIZE_OVERFLOW**: per-envp byte-count + per-uevent SEQNUM arithmetic uses checked operators
- **KERNEXEC**: per-ktype + per-kset dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook `security_kobject_uevent` (already standard upstream) — per-uevent emit gate.
- Default useful GR-RBAC policy: deny custom kobject-creation outside gradm-marked `kernel_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

