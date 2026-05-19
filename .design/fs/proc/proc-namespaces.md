# Tier-3: fs/proc/proc-namespaces — /proc/<pid>/ns/* + setns(2) infrastructure

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/proc/namespaces.c
  - kernel/nsproxy.c
  - kernel/pid_namespace.c
  - kernel/user_namespace.c
  - kernel/utsname.c
  - ipc/namespace.c
  - fs/namespace.c
  - net/core/net_namespace.c
  - kernel/cgroup/namespace.c
  - kernel/time/namespace.c
  - include/linux/proc_ns.h
  - include/uapi/linux/nsfs.h
-->

## Summary
Tier-3 design for the `/proc/<pid>/ns/*` symlink infrastructure + the `setns(2)` syscall path. Per-task namespace symlinks (`mnt`, `uts`, `ipc`, `pid`, `pid_for_children`, `net`, `user`, `cgroup`, `time`, `time_for_children`) point to per-namespace anonymous inodes; opening one returns an `nsfd` that can be passed to `setns(2)` to switch the calling task into that namespace.

This is the cornerstone of every container runtime (Docker, containerd, runc, podman, systemd-nspawn) — `unshare(2) + setns(2) + clone(2) CLONE_NEW*` are how containers are constructed. The `/proc/<pid>/ns/*` interface is also how `nsenter` works.

Sub-tier-3 of `fs/proc/00-overview.md`. Pairs with all the per-namespace subsystems:
- mount-ns: `fs/namespace.c`
- UTS-ns: `kernel/utsname.c`
- IPC-ns: `ipc/namespace.c`
- PID-ns: `kernel/pid_namespace.c`
- net-ns: `net/core/net_namespace.c`
- user-ns: `kernel/user_namespace.c`
- cgroup-ns: `kernel/cgroup/namespace.c`
- time-ns: `kernel/time/namespace.c`

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| /proc/<pid>/ns/* dir + symlinks | `fs/proc/namespaces.c` |
| Per-task namespace proxy | `kernel/nsproxy.c` |
| PID namespace | `kernel/pid_namespace.c` |
| User namespace | `kernel/user_namespace.c` |
| UTS namespace | `kernel/utsname.c` |
| IPC namespace | `ipc/namespace.c` |
| Mount namespace | `fs/namespace.c` |
| Network namespace | `net/core/net_namespace.c` |
| Cgroup namespace | `kernel/cgroup/namespace.c` |
| Time namespace | `kernel/time/namespace.c` |
| Public API | `include/linux/proc_ns.h` |
| UAPI: NS_GET_USERNS / NS_GET_PARENT / NS_GET_NSTYPE / NS_GET_OWNER_UID | `include/uapi/linux/nsfs.h` |

## Compatibility contract

### `/proc/<pid>/ns/*` symlinks

For each task and each namespace type, a symlink:
```
$ ls -l /proc/$$/ns/
lrwxrwxrwx 1 root root 0 ... cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 ... ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 ... mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 ... net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 ... pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 ... pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 ... time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 ... time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 ... user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 ... uts -> 'uts:[4026531838]'
```

Each symlink-target has format `<type>:[<inode>]` where `<inode>` is the anonymous nsfs inode. Tasks in the same namespace share inode (so `readlink(/proc/A/ns/X) == readlink(/proc/B/ns/X)` iff A,B in same X-namespace).

Format byte-identical so `nsenter`, `lsns`, container-aware tools work.

### `pid_for_children` and `time_for_children` (special)

Two special symlinks:
- `pid_for_children` — namespace into which children created via `clone()` will be put; differs from `pid` after `unshare(CLONE_NEWPID)` until the next exec
- `time_for_children` — analogous for time-namespace

Identical semantics.

### `setns(2)` semantics

Per `man 2 setns`:
```c
int setns(int fd, int nstype);
```

`fd` from `open("/proc/<pid>/ns/<type>")`; `nstype` is `0` (any) or `CLONE_NEWNS / NEWUTS / NEWIPC / NEWNET / NEWPID / NEWUSER / NEWCGROUP / NEWTIME`.

After successful setns, calling task's namespace for that type is replaced. Per-namespace-type permission requirements:
- `CLONE_NEWPID` / `NEWUTS` / `NEWIPC` / `NEWNET` / `NEWCGROUP`: requires CAP_SYS_ADMIN in target user-namespace
- `CLONE_NEWNS`: requires CAP_SYS_ADMIN + CAP_SYS_CHROOT
- `CLONE_NEWUSER`: target must be a descendant or ancestor; specific rules per `man 7 user_namespaces`
- `CLONE_NEWTIME`: only valid via `time_for_children` (can't switch own time-ns)

Identical permission gate.

### `nsfs.h` UAPI: NS_GET_* ioctls on nsfds

```c
#define NS_GET_USERNS       _IO(NSIO, 0x1)   /* return userns owning this ns */
#define NS_GET_PARENT       _IO(NSIO, 0x2)   /* return parent ns (PID + user only) */
#define NS_GET_NSTYPE       _IO(NSIO, 0x3)   /* return CLONE_NEW* type */
#define NS_GET_OWNER_UID    _IO(NSIO, 0x4)   /* return owning UID (user-ns only) */
#define NS_GET_MNTNS_ID     _IO(NSIO, 0x5)
#define NS_GET_PID_FROM_PIDNS _IOR(NSIO, 0x6, int)
#define NS_GET_TGID_FROM_PIDNS _IOR(NSIO, 0x7, int)
#define NS_GET_PID_IN_PIDNS _IOR(NSIO, 0x8, int)
#define NS_GET_TGID_IN_PIDNS _IOR(NSIO, 0x9, int)
```

Identical UAPI.

### `unshare(2)` semantics

Per `man 2 unshare`:
```c
int unshare(int flags);
```

Creates new namespace(s) for calling task per `flags = CLONE_NEW*`. Implementation chains:
1. Allocate new per-type namespace(s)
2. Reattach calling task's `nsproxy` to new namespace
3. For PID-ns: only affects future `clone()`-children (unshare doesn't change own PID); see `pid_for_children`

Identical semantics.

### `clone3(2)` `CLONE_NEW*` flags

Equivalent to `unshare + clone` atomic combination. Per `man 2 clone3`. Identical permission requirements.

### `struct nsproxy`

Per-task struct holding pointers to all namespace types:
```c
struct nsproxy {
    refcount_t count;
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns_for_children;
    struct net *net_ns;
    struct time_namespace *time_ns;
    struct time_namespace *time_ns_for_children;
    struct cgroup_namespace *cgroup_ns;
};
```

Layout-byte-identical for first cache-line.

### Per-namespace `proc_ns_operations`

Each namespace type registers:
```c
struct proc_ns_operations {
    const char *name;
    const char *real_name;
    int type;                                /* CLONE_NEW* */
    struct ns_common *(*get)(struct task_struct *task);
    void (*put)(struct ns_common *ns);
    int (*install)(struct nsset *nsset, struct ns_common *ns);
    struct user_namespace *(*owner)(struct ns_common *ns);
    struct ns_common *(*get_parent)(struct ns_common *ns);
};
```

Registry in `fs/proc/namespaces.c` per upstream's static array. Identical layout.

### Hierarchy: PID + user namespaces

PID-ns and user-ns form trees; per `NS_GET_PARENT` returns parent. Other namespace types are flat (no hierarchy).

### Time-namespace virtual offsets

Per-time-namespace `offsets` (per CLOCK_MONOTONIC + CLOCK_BOOTTIME): tasks in time-ns see `clock_gettime(MONOTONIC)` values offset by per-ns offset. Set via `/proc/<pid>/timens_offsets` (read-only after first open).

Identical semantics.

## Requirements

- REQ-1: 10 per-task `/proc/<pid>/ns/<type>` symlinks (cgroup/ipc/mnt/net/pid/pid_for_children/time/time_for_children/user/uts) format byte-identical.
- REQ-2: Symlink target format `<type>:[<inode>]` byte-identical; per-namespace-type unique inode.
- REQ-3: `pid_for_children` + `time_for_children` special symlinks; differ from `pid`/`time` after unshare.
- REQ-4: `setns(2)` syscall with per-type permission requirements byte-identical.
- REQ-5: `nsfs.h` UAPI ioctls (NS_GET_USERNS / PARENT / NSTYPE / OWNER_UID / MNTNS_ID / PID_FROM_PIDNS / TGID_FROM_PIDNS / PID_IN_PIDNS / TGID_IN_PIDNS) byte-identical.
- REQ-6: `unshare(2)` per CLONE_NEW* flag byte-identical effect.
- REQ-7: `clone3(2)` with CLONE_NEW* atomic create-and-fork byte-identical.
- REQ-8: `struct nsproxy` first-cache-line layout-equivalent.
- REQ-9: Per-namespace `proc_ns_operations` static registry; identical entries.
- REQ-10: PID-ns + user-ns hierarchy: `NS_GET_PARENT` ioctl returns parent; root has no parent (returns ENOENT).
- REQ-11: Time-namespace per-clock offsets: `clock_gettime(CLOCK_MONOTONIC)` returns ns-offset value; `timens_offsets` writable once.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `ls -l /proc/$$/ns/` shows 10 symlinks; `readlink` for each returns `<type>:[<inode>]`. (covers REQ-1, REQ-2)
- [ ] AC-2: Same-namespace test: 2 tasks in same netns; `readlink /proc/A/ns/net` == `readlink /proc/B/ns/net`. (covers REQ-2)
- [ ] AC-3: `unshare -n; readlink /proc/self/ns/net` differs from parent's. (covers REQ-2, REQ-6)
- [ ] AC-4: setns test: open `/proc/<pid>/ns/net`; call `setns(fd, CLONE_NEWNET)`; calling task's net-ns now matches that fd. (covers REQ-4)
- [ ] AC-5: pid_for_children test: `unshare --fork --pid bash`; in child, `readlink /proc/self/ns/pid_for_children` differs from `pid`. (covers REQ-3)
- [ ] AC-6: NS_GET_USERNS test: open `/proc/$$/ns/net`; ioctl(fd, NS_GET_USERNS) returns nsfd to owning user-ns. (covers REQ-5)
- [ ] AC-7: NS_GET_PARENT test: open `/proc/$$/ns/pid`; ioctl(fd, NS_GET_PARENT) returns parent-pidns nsfd; on init-pidns, returns ENOENT. (covers REQ-5, REQ-10)
- [ ] AC-8: NS_GET_NSTYPE test: returns CLONE_NEWPID / NEWUSER / NEWNET / etc. matching the nsfd type. (covers REQ-5)
- [ ] AC-9: clone3 with CLONE_NEW* test: `clone3` with `flags = CLONE_NEWPID | CLONE_NEWNET`; child has fresh PID-ns + net-ns. (covers REQ-7)
- [ ] AC-10: Time-ns offset test: `unshare --time --boottime-offset=86400`; `cat /proc/uptime` shows boottime offset by 86400. (covers REQ-11)
- [ ] AC-11: setns permission test: non-root attempts `setns(fd, CLONE_NEWPID)` → EPERM. (covers REQ-4)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::fs::proc::namespaces::ProcNs` — `/proc/<pid>/ns/` directory generator
- `kernel::fs::proc::namespaces::ProcNsLink` — per-type symlink generator
- `kernel::fs::proc::namespaces::Registry` — per-type `proc_ns_operations` registry
- `kernel::ns::nsproxy::NsProxy` — `struct nsproxy` per-task wrapper
- `kernel::ns::nsproxy::Setns` — `setns(2)` syscall path
- `kernel::ns::nsproxy::Unshare` — `unshare(2)` syscall path
- `kernel::ns::nsfs::NsCommon` — `struct ns_common` shared base
- `kernel::ns::nsfs::Ioctl` — NS_GET_* ioctls
- `kernel::ns::pid::PidNamespace` — PID-ns hierarchy
- `kernel::ns::user::UserNamespace` — user-ns hierarchy + capability
- `kernel::ns::uts::UtsNamespace` — UTS-ns
- `kernel::ns::ipc::IpcNamespace`
- `kernel::ns::mnt::MntNamespace`
- `kernel::ns::net::NetNamespace`
- `kernel::ns::cgroup::CgroupNamespace`
- `kernel::ns::time::TimeNamespace` — per-clock offsets

### Locking and concurrency

- **Per-task `task->nsproxy_lock`**: per-task nsproxy mutator
- **Per-namespace-type ref**: refcount-protected via `ns_common`
- **`namespace_sem`** (per-mount-namespace rwsem): mount-ns lifecycle
- **Per-net `net_mutex`**: net-ns lifecycle
- **RCU**: per-task ns lookup hot path RCU-side

### Error handling

- `Err(EINVAL)` — bad nstype / fd not nsfd
- `Err(EPERM)` — capability check failed
- `Err(ENOENT)` — root-ns has no parent
- `Err(ESRCH)` — task gone

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-task /proc/<pid>/ns dir generation (refcount-acquire on task+nsproxy) | `kani::proofs::fs::proc::namespaces::dir_safety` |
| `setns(2)` (per-type permission gate; nsproxy mutation under nsproxy_lock) | `kani::proofs::fs::proc::namespaces::setns_safety` |
| NS_GET_PARENT (no infinite loop on cyclic structure; PID-ns + user-ns chain bounded) | `kani::proofs::fs::proc::namespaces::parent_safety` |
| Time-ns offset application (no overflow on per-clock add) | `kani::proofs::fs::proc::namespaces::time_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-task nsproxy | every namespace pointer in struct nsproxy is non-NULL post-init; refcount ≥ 1 | `kani::proofs::fs::proc::namespaces::nsproxy_invariants` |
| PID-ns + user-ns hierarchy | DAG (no cycles); per-ns parent pointer eventually reaches root | `kani::proofs::fs::proc::namespaces::hierarchy_invariants` |

### Layer 4: Functional correctness (opt-in)

- **setns + permission soundness theorem** via Verus — proves: ∀ setns call, target ns is reachable from caller's user-ns AND caller has CAP_SYS_ADMIN in that user-ns; otherwise returns EPERM.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-namespace + per-nsproxy refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-namespace-type slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed namespace state cleared (carries cgroup-paths + UTS-strings + LSM-secctx) | § Default-on configurable off |
| **CONSTIFY** | per-type `proc_ns_operations` instances + registry array `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, CONSTIFY**: see above
- **USERCOPY**: setns + ioctl arg parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: per-namespace inode arithmetic uses checked operators
- **KERNEXEC**: per-type dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_SYS_ADMIN)` already required for setns (per-type).
- LSM hook `security_inode_permission` for /proc/<pid>/ns/* access.
- Default useful GR-RBAC policy: deny `setns()` outside gradm-marked `container_runtime` role; arbitrary setns is a sandbox-escape primitive.
- DoS-prevention: `user.max_user_namespaces` etc. sysctls cap per-user namespace creation rate.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- PAX_USERCOPY: `ns_get_path()` and `ns_get_name()` write into seq-buffer slabs whitelisted for user copy; readlink results for `/proc/<pid>/ns/<type>` cannot leak adjacent `nsproxy` heap content.
- PAX_KERNEXEC: `setns(2)` reassignment of `task_struct::nsproxy` runs under `task_lock()` with WP asserted; an attacker cannot abuse a hostile `ns_common::ops` pointer to redirect into kernel text.
- PAX_RANDKSTACK: the per-syscall stack frame for `setns()` and `unshare()` is re-randomized so ns-pointer races cannot fingerprint kernel-stack layout.
- PAX_REFCOUNT: every `ns_common::count` and `nsproxy::count` is `refcount_t` with saturation; an unbounded `pidfd` / `/proc/<pid>/ns/<type>` open population cannot wrap a namespace refcount and cause UAF.
- PAX_MEMORY_SANITIZE: freed `nsproxy`, `pid_namespace`, `net`, and `user_namespace` slabs are zeroed before reuse; a successor namespace cannot inherit a stale credential pointer through slab reuse.
- PAX_UDEREF: readlink output is written with SMAP/PAN asserted; a faulting `readlink` target cannot pivot into a kernel-mode write.
- PAX_RAP / kCFI: `proc_ns_link_inode_operations`, `proc_ns_dir_operations`, and `nsfs_dentry_operations` are signature-validated; namespace fops swap to bypass `setns()` capability checks is rejected.
- GRKERNSEC_HIDESYM: `ns_common::inum` is the only address-shaped value exposed; the underlying `struct nsproxy` pointer is never serialized; `%pK` is enforced if any per-ns debug field is added.
- GRKERNSEC_DMESG: `setns(2)` failures (capability, EINVAL on wrong-ns-type) are rate-limited; a fuzzer probing CLONE_NEW* combinations cannot DoS the console.
- `/proc/<pid>/ns/<type>` open enforces `CAP_SYS_PTRACE` against the target task's user namespace AND `PR_SET_DUMPABLE` of the target; setuid victims cannot be probed by unprivileged ptrace-blocked attackers.
- `setns(fd, nstype)` validates: (a) capability in target user-ns, (b) `nstype==0` or matches `ns_common::ops->type`, (c) PID-ns reassignment forbidden after `fork()` (only affects children), (d) cgroup-ns reassignment requires `CAP_SYS_ADMIN` over both source and target.
- `pidfd_open()` retrieves namespace handles without bypassing `ptrace_may_access(PTRACE_MODE_READ_FSCREDS)`; readlink on `/proc/<pid>/ns/*` for a non-dumpable victim returns the same `EACCES` as `/proc/<pid>/maps`.
- Audit: every `setns()` and `unshare()` is logged with source/target inum, capability set, and the namespace types involved — independent of dmesg.

## Open Questions

(none — namespace + setns infra exhaustively specified by upstream + container-runtime test corpus)

## Out of Scope

- Per-namespace subsystem detail (cross-ref individual subsystem Tier-3s — `kernel/user_namespace.md`, `net/core/net_namespace.md`, etc. once added)
- 32-bit-only paths
- Implementation code
