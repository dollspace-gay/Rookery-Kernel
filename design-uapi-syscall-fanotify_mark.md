---
title: "Tier-5 syscall: fanotify_mark(2) — syscall 301"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`fanotify_mark(2)` installs, removes, or modifies a mark on a filesystem object referenced by a fanotify group fd. A mark is the unit of subscription: it pins a `(group, object, event-mask)` triple plus an `ignored_mask` for sub-tree filtering. The object can be (1) an inode (the default), (2) a mount (`FAN_MARK_MOUNT`), or (3) an entire filesystem (`FAN_MARK_FILESYSTEM` — superblock-level). Marks form the routing fabric of fanotify: when an event fires on an inode `i`, the kernel walks `i.fsnotify_marks`, then the containing mount's marks, then the superblock's marks, and dispatches a copy of the event to every matching group's queue. fanotify_mark drives that routing table. Critical for: per-subtree EDR coverage, container-scope filesystem audit, cgroup-bound build-cache invalidation.

This Tier-5 covers the userspace ABI of syscall 301. Group creation lives in `fanotify_init.md`.

### Acceptance Criteria

- [ ] AC-1: `fanotify_mark(fd, FAN_MARK_ADD, FAN_OPEN, AT_FDCWD, "/tmp")` returns 0; subsequent `open("/tmp/x")` emits event.
- [ ] AC-2: `FAN_MARK_ADD` + `FAN_MARK_REMOVE` ⟹ `-EINVAL`.
- [ ] AC-3: `FAN_MARK_MOUNT` + `FAN_MARK_FILESYSTEM` ⟹ `-EINVAL`.
- [ ] AC-4: `FAN_MARK_MOUNT` from non-`CAP_SYS_ADMIN` user ⟹ `-EPERM`.
- [ ] AC-5: `FAN_OPEN_PERM` on `FAN_CLASS_NOTIF` group ⟹ `-EINVAL`.
- [ ] AC-6: `FAN_OPEN_PERM` with `FAN_MARK_MOUNT` ⟹ `-EINVAL`.
- [ ] AC-7: `FAN_MARK_ADD` on non-existent path ⟹ `-ENOENT`.
- [ ] AC-8: `FAN_MARK_REMOVE` on absent mark ⟹ `-ENOENT`.
- [ ] AC-9: `FAN_MARK_FLUSH` removes all marks of chosen object-type; subsequent events on previously-marked objects produce nothing.
- [ ] AC-10: `FAN_MARK_DONT_FOLLOW` on a symlink: mark attaches to the symlink itself.
- [ ] AC-11: `FAN_MARK_ONLYDIR` on a regular file ⟹ `-ENOTDIR`.
- [ ] AC-12: per-uid `UCOUNT_FANOTIFY_MARKS` exceeded ⟹ `-ENOSPC`.
- [ ] AC-13: Mark target outside caller's userns ⟹ `-EXDEV`.
- [ ] AC-14: `FAN_MARK_FILESYSTEM` on tmpfs (no `FS_ALLOW_FSNOTIFY`) ⟹ `-ENODEV` or `-EOPNOTSUPP` depending on fs.
- [ ] AC-15: `pathname` ⟹ `-EFAULT` on bad user pointer.
- [ ] AC-16: `FAN_MARK_EVICTABLE` mark survives normal operation but may be evicted under shrinker pressure.

### Architecture

Rookery surface in `kernel/notify/fanotify/syscall.rs`:

```rust
bitflags! {
    pub struct FanotifyMarkFlags: u32 {
        const ADD                   = 0x00000001;
        const REMOVE                = 0x00000002;
        const DONT_FOLLOW           = 0x00000004;
        const ONLYDIR               = 0x00000008;
        const MOUNT                 = 0x00000010;
        const IGNORED_MASK          = 0x00000020;
        const IGNORED_SURV_MODIFY   = 0x00000040;
        const FLUSH                 = 0x00000080;
        const FILESYSTEM            = 0x00000100;
        const EVICTABLE             = 0x00000200;
        const IGNORE                = 0x00000400;
    }
}

pub enum FanotifyMarkObject {
    Inode(VfsInodeRef),
    Mount(VfsMountRef),
    Superblock(SuperblockRef),
}

pub struct FanotifyMark {
    pub group:    Weak<FanotifyGroup>,
    pub object:   FanotifyMarkObject,
    pub mask:     u64,
    pub ignored:  u64,
    pub flags:    FanotifyMarkFlags,
    pub ucount:   UCountSlot,
}
```

`Fanotify::fanotify_mark(fd, flags, mask, dirfd, pathname) -> isize`:
1. let group = Fanotify::group_from_fd(fd)?;     // EBADF
2. let f = FanotifyMarkFlags::from_bits(flags).ok_or(-EINVAL)?;
3. Fanotify::validate_action_exclusive(f)?;      // EINVAL
4. Fanotify::validate_object_exclusive(f)?;      // EINVAL
5. Fanotify::validate_mask_for_class(mask, &group, f)?;  // EINVAL
6. Fanotify::check_capability_for_object(&group, f)?;    // EPERM
7. if f.contains(FanotifyMarkFlags::FLUSH) {
     return Fanotify::flush(&group, f);
   }
8. let path = Fanotify::resolve_path(dirfd, pathname, &f)?;  // ENOENT/EFAULT/ELOOP/ENOTDIR
9. Fanotify::enforce_userns(&group, &path)?;     // EXDEV
10. let obj = Fanotify::object_from_path(&path, f)?;
11. match action(f) {
12.   Add    => Fanotify::add_mark(&group, obj, mask, f),
13.   Remove => Fanotify::remove_mark(&group, obj, mask, f),
14. }

`Fanotify::add_mark(group, obj, mask, flags) -> isize`:
1. if let Some(m) = group.find_mark(&obj) { if flags.contains(IGNORED_MASK) { m.ignored |= mask; } else { m.mask |= mask; } return Ok(0); }
2. let ucount = group.alloc_ucount_for_mark()?;       // ENOSPC
3. let mut m = FanotifyMark { group: Arc::downgrade(group), object: obj, mask: 0, ignored: 0, flags, ucount };
4. if flags.contains(IGNORED_MASK) { m.ignored = mask; } else { m.mask = mask; }
5. obj.attach_mark(&m); group.marks.insert(m); Ok(0)

### Out of Scope

- `fanotify_init.md` Tier-5 — group creation
- `uapi/headers/fanotify.md` Tier-5 — read-frame, info-record layout, verdict write path
- `fs/notify/fanotify.md` Tier-3 — mark / group / per-class queue lifecycle, fsnotify_marks list walk
- `fs/notify/fsnotify.md` Tier-3 — generic fsnotify dispatcher
- Implementation code

### signature

```c
int fanotify_mark(int fanotify_fd, unsigned int flags, uint64_t mask,
                  int dirfd, const char *pathname);
```

Rust ABI shim:

```rust
pub fn sys_fanotify_mark(
    fanotify_fd: i32,
    flags: u32,
    mask: u64,
    dirfd: i32,
    pathname: UserPtr<u8>,
) -> isize;
```

Syscall number: **301** (x86_64).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `fanotify_fd` | `i32` | IN | fd returned by `fanotify_init(2)` |
| `flags` | `u32` | IN | bitmask: action (`FAN_MARK_ADD` / `_REMOVE` / `_FLUSH`), object-type (`FAN_MARK_MOUNT` / `_FILESYSTEM` / `_INODE` default), modifiers (`FAN_MARK_DONT_FOLLOW`, `FAN_MARK_ONLYDIR`, `FAN_MARK_IGNORED_MASK`, `FAN_MARK_IGNORED_SURV_MODIFY`, `FAN_MARK_EVICTABLE`, `FAN_MARK_IGNORE`) |
| `mask` | `u64` | IN | event mask (`FAN_ACCESS`, `FAN_MODIFY`, `FAN_OPEN`, `FAN_CLOSE_WRITE`, `FAN_CLOSE_NOWRITE`, `FAN_OPEN_EXEC`, `FAN_*_PERM`, `FAN_CREATE`, `FAN_DELETE`, `FAN_MOVED_FROM`, `FAN_MOVED_TO`, `FAN_RENAME`, `FAN_ATTRIB`, `FAN_FS_ERROR`, `FAN_ONDIR`, `FAN_EVENT_ON_CHILD`) |
| `dirfd` | `i32` | IN | base directory fd for `*at`-style path resolution; `AT_FDCWD` for cwd-relative |
| `pathname` | `const char *` | IN | path relative to `dirfd`; `NULL` only valid when `dirfd` is the target fd itself (rare); empty string with `AT_EMPTY_PATH` not supported |

### return

- **Success**: `0`.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EBADF` | `fanotify_fd` invalid or not a fanotify group; `dirfd` invalid for relative path |
| `EINVAL` | unknown bits in `flags`; conflicting action bits (`ADD` + `REMOVE`); conflicting object-type bits; `mask` has bits the group's class cannot deliver (e.g. `FAN_OPEN_PERM` on `FAN_CLASS_NOTIF`); `FAN_MARK_FILESYSTEM` on a fs without `FS_ALLOW_FSNOTIFY`; bad pathname pointer |
| `ENOENT` | `pathname` does not exist (`ADD`); or mark not present (`REMOVE`) |
| `ENOSPC` | per-uid `UCOUNT_FANOTIFY_MARKS` cap reached without `FAN_UNLIMITED_MARKS` |
| `EPERM` | `FAN_MARK_MOUNT` / `FAN_MARK_FILESYSTEM` without `CAP_SYS_ADMIN`; permission-class mask without `CAP_SYS_ADMIN` |
| `EXDEV` | mark target outside the group's user namespace |
| `ENODEV` | filesystem does not support `FAN_MARK_FILESYSTEM` |
| `EOPNOTSUPP` | mask contains bit unsupported by underlying fs |
| `ENOTDIR` | `FAN_MARK_ONLYDIR` but `pathname` is not a directory |
| `EFAULT` | `pathname` faults during copy-in |
| `ELOOP` | symlink loop in resolution; or `FAN_MARK_DONT_FOLLOW` and terminal symlink (with deny semantics) |
| `ENAMETOOLONG` | `pathname` too long |

### abi surface

Action bits (exactly one): `FAN_MARK_ADD = 0x1` (OR-in mask), `FAN_MARK_REMOVE = 0x2` (clear bits; delete mark if mask reaches 0), `FAN_MARK_FLUSH = 0x80` (remove all marks of chosen object-type).

Object-type bits (at most one; default = inode): `FAN_MARK_INODE = 0x0` (default), `FAN_MARK_MOUNT = 0x10`, `FAN_MARK_FILESYSTEM = 0x100`.

Modifiers: `FAN_MARK_DONT_FOLLOW = 0x4` (no terminal-symlink follow), `FAN_MARK_ONLYDIR = 0x8` (refuse non-dir), `FAN_MARK_IGNORED_MASK = 0x20` (apply to ignored set), `FAN_MARK_IGNORED_SURV_MODIFY = 0x40` (ignored survives), `FAN_MARK_EVICTABLE = 0x200` (shrinker-reclaimable), `FAN_MARK_IGNORE = 0x400` (5.19+ unified).

Event mask bits (selected; full list in `uapi/headers/fanotify.md`):

| Constant | Value | Purpose |
|---|---|---|
| `FAN_ACCESS / MODIFY / ATTRIB` | `0x1 / 0x2 / 0x4` | access / write / metadata |
| `FAN_CLOSE_WRITE / CLOSE_NOWRITE / OPEN` | `0x8 / 0x10 / 0x20` | close-rw / close-ro / open |
| `FAN_MOVED_FROM / MOVED_TO` | `0x40 / 0x80` | rename source / target |
| `FAN_CREATE / DELETE / DELETE_SELF / MOVE_SELF` | `0x100..0x800` | child/self lifecycle |
| `FAN_OPEN_EXEC` | `0x1000` | open for exec |
| `FAN_Q_OVERFLOW` | `0x4000` | event-only sentinel (not settable) |
| `FAN_FS_ERROR` | `0x8000` | fs-error (super only) |
| `FAN_OPEN_PERM / ACCESS_PERM / OPEN_EXEC_PERM` | `0x10000..0x40000` | permission gates |
| `FAN_RENAME` | `0x10000000` | rename with target fid |
| `FAN_ONDIR / EVENT_ON_CHILD` | `0x40000000 / 0x08000000` | dir-event / child-event scoping |

### compatibility contract

REQ-1: `fanotify_fd` must reference a fanotify group; `EBADF` otherwise.

REQ-2: Action bits are mutually exclusive among `{ADD, REMOVE, FLUSH}`; exactly one required.

REQ-3: Object-type bits are mutually exclusive among `{INODE(default), MOUNT, FILESYSTEM}`; at most one.

REQ-4: Permission mask bits (`FAN_*_PERM`) require the group's class to be `FAN_CLASS_CONTENT` or `FAN_CLASS_PRE_CONTENT`. Otherwise `-EINVAL`.

REQ-5: `FAN_MARK_MOUNT` and `FAN_MARK_FILESYSTEM` require `CAP_SYS_ADMIN` in the user namespace owning the mount / superblock.

REQ-6: Mask validation:
- Bits set outside the valid event-mask domain ⟹ `-EINVAL`.
- `FAN_Q_OVERFLOW` not settable ⟹ `-EINVAL` if present.
- Permission bits with `FAN_MARK_MOUNT` ⟹ `-EINVAL` (perm events require inode-level marks for atomicity).

REQ-7: `FAN_MARK_FLUSH` ignores `mask`, `dirfd`, `pathname`; clears all marks of the chosen object-type from the group.

REQ-8: Path resolution:
- `pathname` is resolved relative to `dirfd` using `*at`-family semantics.
- `FAN_MARK_DONT_FOLLOW` ⟹ do not follow terminal symlink (analogous to `AT_SYMLINK_NOFOLLOW`).
- `FAN_MARK_ONLYDIR` ⟹ refuse non-directory target.
- Resolution honors mount-namespace of caller.

REQ-9: User-namespace fence:
- Mark target must be inside `current_user_ns()` (or a descendant). `EXDEV` otherwise.

REQ-10: Per-uid `UCOUNT_FANOTIFY_MARKS`: ADD increments unless `FAN_UNLIMITED_MARKS`; REMOVE decrements; FLUSH decrements by `n` (number removed).

REQ-11: Ignored mask: `FAN_MARK_IGNORED_MASK` ⟹ bits go to ignored-set (suppress delivery); `FAN_MARK_IGNORED_SURV_MODIFY` ⟹ ignored persists across edits; `FAN_MARK_IGNORE` (5.19+) replaces both with unified semantics.

REQ-12: `FAN_MARK_EVICTABLE` permits shrinker eviction; used by AV scanners on large hierarchies.

REQ-13: Idempotence: ADD on existing mark OR-in bits (no error); REMOVE on absent ⟹ `-ENOENT`; REMOVE driving mask to 0 destroys mark.

REQ-14: `dirfd == AT_FDCWD` and `pathname == "/"` valid for filesystem-root marks.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `action_exactly_one` | INVARIANT | exactly one of `ADD / REMOVE / FLUSH` |
| `object_at_most_one` | INVARIANT | at most one of `MOUNT / FILESYSTEM` |
| `perm_mask_class_match` | INVARIANT | `FAN_*_PERM` ⟹ group class is `CONTENT` or `PRE_CONTENT` |
| `perm_mask_requires_inode` | INVARIANT | `FAN_*_PERM` cannot use `FAN_MARK_MOUNT` |
| `ucount_balanced` | INVARIANT | `ADD` increments, `REMOVE`/`FLUSH` decrement, `ADD` on existing does not double-count |
| `userns_fence` | INVARIANT | mark target inside `current_user_ns()` |
| `cap_check_for_mount_fs` | INVARIANT | `MOUNT` / `FILESYSTEM` requires `CAP_SYS_ADMIN` |

### Layer 2: TLA+

`uapi/fanotify_mark.tla`:
- States: `validating`, `cap_check`, `path_resolve`, `userns_check`, `applying`, `done`, `failed`.
- Properties:
  - `safety_action_exclusive` — action bits never overlap.
  - `safety_perm_class_match` — permission masks only on permission groups.
  - `safety_no_priv_escalation` — mount/superblock marks require CAP_SYS_ADMIN.
  - `safety_ucount_paired` — for every successful ADD, the matching REMOVE / FLUSH / group-close decrements UCOUNT.
  - `liveness_terminate` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `fanotify_mark` post(ok, ADD): mark present on object with mask ⊇ requested | `Fanotify::add_mark` |
| `fanotify_mark` post(ok, REMOVE): mark mask ∩ requested == 0 | `Fanotify::remove_mark` |
| `fanotify_mark` post(ok, FLUSH): no marks of chosen object-type belong to group | `Fanotify::flush` |
| `fanotify_mark` post(err): no ucount delta, no object-side mark mutation | `Fanotify::fanotify_mark` |

### Layer 4: Verus/Creusot functional

Per-`fanotify_mark(2)` / `fanotify(7)` man pages, `Documentation/admin-guide/filesystem-monitoring.rst`, and per-LTP `testcases/kernel/syscalls/fanotify/fanotify0[1-9].c` round-trip; semantic equivalence with libcap-ng wrappers.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

fanotify_mark reinforcement:

- **CAP_SYS_ADMIN gate for mount / superblock marks** — defense against unprivileged listener subscribing to events of objects it does not own.
- **Permission-mask × class match** — defense against perm-class semantics smuggled into notification-class groups.
- **UCount-based per-uid mark cap** — defense against per-user mark-table flood.
- **Userns fence** — defense against cross-namespace event observation.
- **Path resolution honors mount-namespace** — defense against namespace-confusion.
- **Action / object-type mutual exclusion** — defense against partial-state on contradictory flags.
- **FLUSH bounded by per-type set** — defense against accidental cross-type wipe.

### grsecurity/pax-style reinforcement

- **PaX UDEREF on `pathname` copy-in** — `getname()` runs under SMAP/UDEREF; the kernel never directly dereferences a user pointer for this argument. PAX_USERCOPY_HARDEN bounds the copy via whitelisted slab; refuse beyond `PATH_MAX`.
- **GRKERNSEC_HARDEN_USERFAULTFD-style opt-in for permission masks** — `FAN_*_PERM` marks can indefinitely stall arbitrary processes. Under hardened policy, permission masks require `CAP_SYS_ADMIN` in `init_user_ns` (never a delegated cap from a child userns), plus the per-mount-ns ACK sysctl `fs.fanotify.permission_class_allowed = 1`. Mirrors `vm.unprivileged_userfaultfd = 0`.
- **CAP_SYS_ADMIN required (init_userns) for FAN_MARK_MOUNT / FAN_MARK_FILESYSTEM** — even when the calling userns has CAP_SYS_ADMIN via clone-with-uid-map, deny unless the cap is also held in the initial userns; mount and superblock marks affect every process touching the object.
- **PAX_REFCOUNT on mark refcounts** — defense against per-refcount-overflow UAF on long-lived marks accumulated by AV-scanner groups across billions of ADD/REMOVE pairs.
- **Landlock + GRKERNSEC_CHROOT_FINDTASK as complementary sandbox** — when caller is chrooted, refuse mark installation on objects above the chroot root; fanotify routing through mount/superblock marks would otherwise let a chrooted listener observe events on objects outside its chroot. A process that has called `landlock_restrict_self` to drop `LANDLOCK_ACCESS_FS_*` rights cannot install fanotify marks that would re-expose those rights via event delivery.
- **GRKERNSEC_HIDESYM on `fanotify_event_info_fid` payloads** — `FAN_REPORT_FID` records contain an opaque `file_handle` blob; on ext4 / xfs this is a stable inode-number side-channel. Refuse `FAN_REPORT_FID` event delivery for groups created by an unprivileged uid in a userns shared with privileged processes, unless the marked object is owned by the listener's uid or it holds `CAP_DAC_READ_SEARCH`.
- **Per-uid quota cross-checked with `fs.fanotify.max_user_marks`** — refuse ADD beyond the lower of (global, userns) cap; per-userns sysctls must be tunable to small values for untrusted containers.
- **CAP_LINUX_IMMUTABLE observed on FAN_MARK_FILESYSTEM** — superblock marks on an immutable-by-attribute fs (frozen / r/o-remount) require `CAP_LINUX_IMMUTABLE` for ADD when the mark could perturb writeback (`FAN_ATTRIB`, `FAN_MODIFY`). Bounds AV-scanner interference with frozen storage.
- **FAN_MARK_EVICTABLE mandatory for unprivileged groups** — under hardened policy, marks installed by uids without `CAP_SYS_RESOURCE` are forced `EVICTABLE`; bounds long-term unswappable kernel-memory usage.
- **Audit log on every mount / superblock mark add and on rate-limit churn** — `FAN_MARK_MOUNT` / `FAN_MARK_FILESYSTEM` ADD always logged at `LOGLEVEL_NOTICE`; a process performing > N ADD/REMOVE pairs per second on the same group is logged and throttled (prevents covert-channel use of mark-table churn). Path resolution refuses marks crossing into a peer mount namespace via shared-mount propagation the caller did not originally establish.

