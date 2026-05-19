---
title: "Tier-5 syscall: fanotify_init(2) — syscall 300"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`fanotify_init(2)` allocates a kernel-resident fanotify notification group — an `fsnotify_group` configured with the fanotify ops, a bounded event queue, and a per-class priority slot — and returns a file descriptor on it. fanotify differs structurally from inotify on four axes: (1) it can deliver permission-class events that block the originating syscall until userspace `write(2)`s a verdict; (2) it can deliver events with an open file descriptor of the affected object (`FAN_REPORT_FD`) so the listener can act on the object without resolving a path; (3) it can deliver events keyed by `fanotify_event_info_fid` (filesystem ID + inode handle) for filesystem-scope monitoring across mount namespaces; (4) it can attach marks to mounts and entire filesystems, not just inodes. The two arguments to `fanotify_init` partition a large flag space — `flags` controls per-group behavior (class, blocking, reporting, queue caps) and `event_f_flags` is the `open(2)`-style flag set used to construct the per-event `struct file` (when `FAN_REPORT_FD` semantics apply). Critical for: AV / EDR scanners, container security policy enforcement, filesystem audit, build-system change-trigger across mount namespaces.

This Tier-5 covers the userspace ABI of syscall 300 — group creation and class selection. Mark installation lives in `fanotify_mark.md`.

### Acceptance Criteria

- [ ] AC-1: `fanotify_init(FAN_CLASS_NOTIF, O_RDONLY)` from `CAP_SYS_ADMIN` returns positive fd.
- [ ] AC-2: `fanotify_init(0, O_RDONLY)` (no class) ⟹ `-EINVAL`.
- [ ] AC-3: `fanotify_init(FAN_CLASS_NOTIF | FAN_CLASS_CONTENT, O_RDONLY)` (two classes) ⟹ `-EINVAL`.
- [ ] AC-4: `fanotify_init(FAN_CLASS_CONTENT, O_RDONLY)` from unprivileged user ⟹ `-EPERM`.
- [ ] AC-5: `fanotify_init(FAN_CLASS_NOTIF | FAN_CLOEXEC, O_RDONLY)`: `fcntl(fd, F_GETFD) & FD_CLOEXEC`.
- [ ] AC-6: `fanotify_init(FAN_CLASS_NOTIF | FAN_NONBLOCK, O_RDONLY)`: `fcntl(fd, F_GETFL) & O_NONBLOCK`.
- [ ] AC-7: `fanotify_init(FAN_CLASS_NOTIF | FAN_REPORT_NAME, O_RDONLY)` without `FAN_REPORT_DIR_FID` ⟹ `-EINVAL`.
- [ ] AC-8: `fanotify_init(FAN_CLASS_NOTIF | FAN_UNLIMITED_QUEUE, O_RDONLY)` from unprivileged user ⟹ `-EPERM`.
- [ ] AC-9: `event_f_flags = O_DIRECTORY` ⟹ `-EINVAL` (not in allowed set).
- [ ] AC-10: per-uid `fs.fanotify.max_user_groups` exceeded ⟹ `-EMFILE`.
- [ ] AC-11: `RLIMIT_NOFILE` exhausted ⟹ `-EMFILE`.
- [ ] AC-12: `close(fd)` decrements `UCOUNT_FANOTIFY_GROUPS`.
- [ ] AC-13: After group close with outstanding permission events: each is auto-denied (kernel falls back to allow per legacy doc — see Open Questions).
- [ ] AC-14: Group fd visible as `anon_inode:[fanotify]` in `/proc/<pid>/fd/`.

### Architecture

Rookery surface in `kernel/notify/fanotify/syscall.rs`:

```rust
bitflags! {
    pub struct FanotifyInitFlags: u32 {
        const CLOEXEC          = 0x00000001;
        const NONBLOCK         = 0x00000002;
        const CLASS_NOTIF      = 0x00000000;
        const CLASS_CONTENT    = 0x00000004;
        const CLASS_PRE_CONTENT= 0x00000008;
        const UNLIMITED_QUEUE  = 0x00000010;
        const UNLIMITED_MARKS  = 0x00000020;
        const ENABLE_AUDIT     = 0x00000040;
        const REPORT_PIDFD     = 0x00000080;
        const REPORT_TID       = 0x00000100;
        const REPORT_FID       = 0x00000200;
        const REPORT_DIR_FID   = 0x00000400;
        const REPORT_NAME      = 0x00000800;
        const REPORT_TARGET_FID= 0x00001000;
    }
}

pub struct FanotifyGroup {
    pub queue:        SpinLockedDeque<FanotifyEvent>,
    pub marks:        RwLockedSet<FanotifyMark>,
    pub class:        FanotifyClass,        // Notif | Content | PreContent
    pub priority:     u8,
    pub max_events:   u32,
    pub max_marks:    u32,
    pub event_f_flags:u32,
    pub flags:        FanotifyInitFlags,
    pub ucounts:      UCountsRef,
    pub wq:           WaitQueue,
    pub overflow:     AtomicBool,
}
```

`Fanotify::fanotify_init(flags, event_f_flags) -> isize`:
1. let f = FanotifyInitFlags::from_bits(flags).ok_or(-EINVAL)?;
2. Fanotify::validate_class_exclusive(f)?;        // EINVAL on 0 or >=2 class bits
3. Fanotify::validate_reporting_deps(f)?;         // REPORT_NAME ⟹ REPORT_DIR_FID, etc.
4. Fanotify::validate_event_f_flags(event_f_flags)?;
5. Fanotify::check_capability_for_flags(f)?;     // EPERM for CONTENT/PRE_CONTENT/UNLIMITED_*/ENABLE_AUDIT
6. let ucounts = inc_ucount(UCOUNT_FANOTIFY_GROUPS).ok_or(-EMFILE)?;
7. let group = Arc::new(FanotifyGroup {
     queue: SpinLockedDeque::new(),
     marks: RwLockedSet::new(),
     class: FanotifyClass::from(f),
     priority: priority_from_class(f),
     max_events: if f.contains(FanotifyInitFlags::UNLIMITED_QUEUE) { u32::MAX } else { sysctl_max_queued_events() },
     max_marks: if f.contains(FanotifyInitFlags::UNLIMITED_MARKS) { u32::MAX } else { sysctl_max_user_marks() },
     event_f_flags,
     flags: f,
     ucounts,
     wq: WaitQueue::new(),
     overflow: AtomicBool::new(false),
   });
8. let open_flags = O_RDONLY
     | if f.contains(FanotifyInitFlags::CLOEXEC) { O_CLOEXEC } else { 0 }
     | if f.contains(FanotifyInitFlags::NONBLOCK) { O_NONBLOCK } else { 0 };
9. anon_inode_getfd("[fanotify]", &FANOTIFY_FOPS, group, open_flags).map(|fd| fd as isize)

`FANOTIFY_FOPS`:
1. read     ⟹ `Fanotify::read` (dequeue metadata + info records; construct per-event fd if class delivers fds).
2. write    ⟹ `Fanotify::write` (verdict-reply for permission events: `FAN_ALLOW` / `FAN_DENY` + fd).
3. poll     ⟹ `Fanotify::poll`.
4. unlocked_ioctl ⟹ `FIONREAD`.
5. release  ⟹ drain queue (auto-deny outstanding perm events), detach marks, dec ucounts, free group.
6. show_fdinfo ⟹ enumerate marks as `fanotify mnt_id:%x mflags:%x mask:%x ignored_mask:%x`.

### Out of Scope

- `fanotify_mark.md` Tier-5 — mark installation
- `uapi/headers/fanotify.md` Tier-5 — read-frame, `fanotify_event_metadata`, `fanotify_event_info_*` record layout, verdict write path
- `fs/notify/fanotify.md` Tier-3 — `fsnotify_group` / `fanotify_mark` / per-class queue lifecycle
- Implementation code

### signature

```c
int fanotify_init(unsigned int flags, unsigned int event_f_flags);
```

Rust ABI shim:

```rust
pub fn sys_fanotify_init(flags: u32, event_f_flags: u32) -> isize;
```

Syscall number: **300** (x86_64).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `flags` | `u32` | IN | per-group: class (`FAN_CLASS_*`), blocking (`FAN_NONBLOCK`), cloexec (`FAN_CLOEXEC`), reporting (`FAN_REPORT_*`), unlimited variants (`FAN_UNLIMITED_QUEUE`, `FAN_UNLIMITED_MARKS`) |
| `event_f_flags` | `u32` | IN | `open(2)` flag set used by kernel when constructing the per-event `struct file` exposed to the listener; e.g. `O_RDONLY`, `O_RDWR`, `O_LARGEFILE`, `O_CLOEXEC` |

### return

- **Success**: non-negative file descriptor referencing the new fanotify group.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | unknown bit in `flags` or `event_f_flags`; mutually exclusive class bits; `FAN_REPORT_FID` combined with a class that delivers FDs (i.e. without `FAN_REPORT_NAME` it must not also set `FAN_CLASS_CONTENT` / `FAN_CLASS_PRE_CONTENT`) |
| `EPERM` | listener lacks `CAP_SYS_ADMIN` for permission-class group (or `CAP_SYS_ADMIN` requirement bypassed only under fanotify-unprivileged knob) |
| `EMFILE` | per-process fd table full (`RLIMIT_NOFILE`); or per-uid fanotify-group cap exceeded |
| `ENFILE` | system-wide fd table full |
| `ENOMEM` | cannot allocate `fsnotify_group` |
| `ENOSYS` | `CONFIG_FANOTIFY` disabled |

### abi surface

Class bits — exactly one required (mutually exclusive):

| Constant | Value | Purpose |
|---|---|---|
| `FAN_CLASS_NOTIF` | `0x00000000` | notification only |
| `FAN_CLASS_CONTENT` | `0x00000004` | content-access permission gating |
| `FAN_CLASS_PRE_CONTENT` | `0x00000008` | pre-content (hierarchical sync / scanner) permission |

File-descriptor / blocking modifiers:

| Constant | Value | Purpose |
|---|---|---|
| `FAN_CLOEXEC` | `0x00000001` | close-on-exec on group fd |
| `FAN_NONBLOCK` | `0x00000002` | non-blocking reads |
| `FAN_UNLIMITED_QUEUE` | `0x00000010` | bypass default queue cap (requires `CAP_SYS_ADMIN`) |
| `FAN_UNLIMITED_MARKS` | `0x00000020` | bypass default per-group mark cap (requires `CAP_SYS_ADMIN`) |
| `FAN_ENABLE_AUDIT` | `0x00000040` | log permission decisions to audit subsystem (requires `CAP_AUDIT_WRITE`) |

Reporting modifiers (deliver `fanotify_event_info_*` records after the fixed `fanotify_event_metadata` header):

| Constant | Value | Purpose |
|---|---|---|
| `FAN_REPORT_TID` | `0x00000100` | report originating TID (not just PID) |
| `FAN_REPORT_FID` | `0x00000200` | report filesystem-id + file-handle (no FD) |
| `FAN_REPORT_DIR_FID` | `0x00000400` | also report parent-dir handle |
| `FAN_REPORT_NAME` | `0x00000800` | also report basename (requires `FAN_REPORT_DIR_FID`) |
| `FAN_REPORT_PIDFD` | `0x00000080` | report originator as pidfd |
| `FAN_REPORT_TARGET_FID` | `0x00001000` | for rename, report target handle |

Per-uid / per-group defaults (sysctl):
- `fs.fanotify.max_user_groups` — per-uid group cap (default `128`).
- `fs.fanotify.max_user_marks` — per-uid mark cap (default `8192`).
- `fs.fanotify.max_queued_events` — default queue depth (`16384`).

### compatibility contract

REQ-1: Exactly one class bit must be set in `flags`. Zero or two class bits ⟹ `-EINVAL`.

REQ-2: `FAN_CLASS_CONTENT` and `FAN_CLASS_PRE_CONTENT` enable permission events (`FAN_OPEN_PERM`, `FAN_ACCESS_PERM`, `FAN_OPEN_EXEC_PERM`) and require `CAP_SYS_ADMIN` unless `sysctl fs.fanotify.fanotify_unprivileged = 1` (notification-class only) is in force. Permission-class always requires `CAP_SYS_ADMIN`.

REQ-3: `FAN_UNLIMITED_QUEUE` / `FAN_UNLIMITED_MARKS` require `CAP_SYS_ADMIN`.

REQ-4: `FAN_REPORT_NAME` requires `FAN_REPORT_DIR_FID`. `FAN_REPORT_TARGET_FID` requires `FAN_REPORT_FID | FAN_REPORT_DIR_FID | FAN_REPORT_NAME`.

REQ-5: `event_f_flags` validation: only `O_RDONLY | O_WRONLY | O_RDWR` access mode plus `O_LARGEFILE | O_CLOEXEC | O_APPEND | O_NONBLOCK | O_SYNC` and `O_NOATIME` allowed; anything else `-EINVAL`.

REQ-6: Group priority — kernel maintains per-mount/per-fsid lists of marks ordered by class priority `PRE_CONTENT > CONTENT > NOTIF`. Multiple permission-class groups on the same object see events in priority order; verdicts compose conservatively (any DENY ⟹ DENY).

REQ-7: Returned fd:
- read frame is `struct fanotify_event_metadata` followed by zero or more variable-length `fanotify_event_info_*` records.
- `O_CLOEXEC` set iff `FAN_CLOEXEC`.
- `O_NONBLOCK` set iff `FAN_NONBLOCK`.
- Poll-able: `POLLIN ⟺ event queue non-empty`.

REQ-8: Per-uid accounting:
- `inc_ucount(UCOUNT_FANOTIFY_GROUPS)`; failure ⟹ `-EMFILE`.
- Mark accounting (`UCOUNT_FANOTIFY_MARKS`) charged in `fanotify_mark`.

REQ-9: Lifetime:
- `close(2)` ⟹ `fanotify_release`: drain queue (denying any outstanding permission event), detach marks, decrement ucounts, free group.
- `fork(2)`: shared file; queue shared.
- `execve(2)`: closed iff `O_CLOEXEC`.

REQ-10: User-namespace scope:
- `ucounts` rooted at `current_user_ns()`.
- Per-namespace caps independent.
- Marks on filesystems / mounts outside the userns are rejected (`-EXDEV` in `fanotify_mark`).

REQ-11: Queue overflow:
- A single `FAN_Q_OVERFLOW` metadata event with `fd == FAN_NOFD`, `mask == FAN_Q_OVERFLOW` is enqueued; subsequent events dropped until queue drained.

REQ-12: `fanotify_unprivileged` sysctl:
- `0`: only `CAP_SYS_ADMIN` may create any group (default historically).
- `1`: unprivileged users may create `FAN_CLASS_NOTIF` groups (with `FAN_REPORT_FID` only — no FD delivery).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `class_exactly_one` | INVARIANT | exactly one of `CLASS_NOTIF / _CONTENT / _PRE_CONTENT` set |
| `report_name_requires_dir_fid` | INVARIANT | `REPORT_NAME ⟹ REPORT_DIR_FID` |
| `unlimited_requires_cap` | INVARIANT | `UNLIMITED_*` requires `CAP_SYS_ADMIN` |
| `content_class_requires_cap` | INVARIANT | `CLASS_CONTENT / PRE_CONTENT` requires `CAP_SYS_ADMIN` |
| `cloexec_propagates` | INVARIANT | `FAN_CLOEXEC ⟹ FD_CLOEXEC` |
| `nonblock_propagates` | INVARIANT | `FAN_NONBLOCK ⟹ O_NONBLOCK` |
| `ucount_balanced` | INVARIANT | inc on init, dec on release |
| `fd_install_atomic` | INVARIANT | fd visible iff group fully initialized |

### Layer 2: TLA+

`uapi/fanotify_init.tla`:
- States: `validating`, `cap_checking`, `accounting_ucount`, `allocating_group`, `installing_fd`, `returned`, `failed`.
- Properties:
  - `safety_ucount_paired_release` — every successful init must be matched by a release.
  - `safety_no_priv_escalation` — `CLASS_CONTENT` without `CAP_SYS_ADMIN` is rejected.
  - `safety_class_exclusive` — class bits mutually exclusive.
  - `safety_no_partial_install` — fd visible iff group fully initialized.
  - `liveness_terminate` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `fanotify_init` post(ok): `0 ≤ ret` ∧ group queue empty ∧ class matches `flags` | `Fanotify::fanotify_init` |
| `fanotify_init` post: group.ucounts increments by 1 | `Fanotify::fanotify_init` |
| `fanotify_init` post(err): no ucount delta, no fd allocated | `Fanotify::fanotify_init` |

### Layer 4: Verus/Creusot functional

Per-`fanotify(7)` / `fanotify_init(2)` man pages and `Documentation/admin-guide/filesystem-monitoring.rst` semantic equivalence. Per-LTP `testcases/kernel/syscalls/fanotify/fanotify01..NN.c` and glibc / libcap-ng wrappers round-trip.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

fanotify_init reinforcement:

- **CLOEXEC default for content-class groups** — defense against perm-class group fd leaked across `execve(2)` to a less-privileged child that then becomes the policy authority.
- **UCount-based per-uid group cap** — defense against per-user fanotify-group flood.
- **UCount-based per-uid mark cap** — defense against per-user mark-table flood.
- **Queue depth bounded** — defense against event-flood OOM.
- **Class-exclusive validation pre-allocation** — defense against partial state on invalid flag bundles.
- **Permission-class CAP_SYS_ADMIN gate** — defense against unprivileged process gating other processes' syscalls.

### grsecurity/pax-style reinforcement

- **PaX UDEREF on syscall entry** — `fanotify_init` takes only scalars, but the entry trampoline still enforces SMAP/UDEREF before dispatching to the body; the kernel never dereferences a user pointer in `fanotify_init` itself.
- **GRKERNSEC_HARDEN_USERFAULTFD-style opt-in for permission-class groups** — `FAN_CLASS_CONTENT` and `FAN_CLASS_PRE_CONTENT` give the listener the power to indefinitely stall arbitrary syscalls in arbitrary processes. Under hardened policy, treat permission-class groups exactly like userfaultfd's `vm.unprivileged_userfaultfd = 0` regime: require `CAP_SYS_ADMIN` always (no `fanotify_unprivileged` escape hatch for content classes), and additionally require an explicit policy ACK via `fs.fanotify.permission_class_allowed = 1` per-mount-namespace, default `0`.
- **CAP_SYS_ADMIN required for FAN_UNLIMITED_QUEUE / FAN_UNLIMITED_MARKS / FAN_ENABLE_AUDIT** — preserved verbatim from upstream; grsec doctrine additionally requires that the cap be present in the **initial** user namespace, not just the current one, since per-userns root with `unprivileged_userns_clone = 1` would otherwise trivially obtain it.
- **PAX_REFCOUNT on `fanotify_group.refcount` and on `fanotify_mark.refcount`** — defense against per-refcount-overflow UAF on long-lived AV-scanner groups that may accumulate billions of mark add/remove cycles.
- **PAX_RANDKSTACK on fanotify_init entry** — fanotify groups are durable, often opened from network-exposed audit/EDR daemons; randomize kernel stack at every group creation.
- **GRKERNSEC_HIDESYM on `/proc/<pid>/fdinfo/<N>` mark entries and on `fanotify_event_info_fid` payloads** — the per-event `fid` info-record exposes filesystem `f_fsid` plus the opaque `file_handle` blob; on filesystems where the handle is derived from inode number plus generation, this is a stable inode-number side-channel. Under `kernel.kptr_restrict ≥ 2`, restrict fdinfo to `CAP_SYS_ADMIN` or same-uid + same-pidns, and forbid `FAN_REPORT_FID` entirely from groups created by unprivileged uids that share a userns with privileged processes.
- **GRKERNSEC_CHROOT_FINDTASK-style namespace fence** — when chroot is active and the chroot policy is set, refuse `fanotify_init` from inside the chroot for permission classes (the listener could otherwise gate syscalls of processes outside its chroot if it later managed to install marks above its root). Landlock (see `landlock_create_ruleset.md`, `landlock_restrict_self.md`) is the **complementary** sandbox primitive: Landlock confines the *issuer*, while fanotify lets the issuer confine others; under hardened policy a process that has called `landlock_restrict_self` is forbidden from subsequently calling `fanotify_init` with a permission class.
- **Per-uid quota cross-checked with `fs.fanotify.max_user_groups` / `max_user_marks`** — even when global caps are large, per-userns sysctls must be tunable to small values for untrusted containers; refuse beyond the lower of (global, userns) cap.
- **CAP_LINUX_IMMUTABLE observed on `FAN_REPORT_TARGET_FID`** — rename-tracking on `S_IMMUTABLE` inodes logged for forensic correlation. Audit log on every permission-class group creation at `LOGLEVEL_NOTICE` with creator uid + creds + pidns + namespace IDs, regardless of `FAN_ENABLE_AUDIT` (listener's audit-write cap is independent of the kernel's audit trail of policy authority).
- **fanotify priority inversion check** — when a higher-priority `PRE_CONTENT` group denies and a lower-priority `CONTENT` group is also installed on the same object, the verdict is always DENY without consulting `CONTENT`; under hardened policy, log the priority dominance for audit. This prevents a malicious lower-priority listener from masking a higher-priority deny by stalling its own verdict (it cannot — but the audit trail must record the decisive group).
- **GRKERNSEC_HIDESYM on `fanotify_event_info_pidfd` and `_tid`** — `FAN_REPORT_PIDFD` / `FAN_REPORT_TID` leak originator identity across pidns boundaries; under hardened policy, replace cross-pidns identities with sentinel `FAN_NOPIDFD` / `0`, and refuse more than `N` events per millisecond from a single mark with an audit-log entry on rate-limit trip.

