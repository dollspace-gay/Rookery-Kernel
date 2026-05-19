# Tier-5 syscall: inotify_init1(2) — syscall 294

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/notify/inotify/inotify_user.c (sys_inotify_init1, sys_inotify_init, inotify_new_group)
  - fs/notify/inotify/inotify_fsnotify.c
  - include/uapi/linux/inotify.h
  - arch/*/include/generated/uapi/asm/unistd_64.h (294)
  - Documentation/filesystems/inotify.rst, man inotify(7), inotify_init1(2)
-->

## Summary

`inotify_init1(2)` allocates a kernel-resident inotify instance — an `fsnotify_group` plus an event queue — and returns a file descriptor on it. Supersedes the flagless `inotify_init(2)` (syscall 253) by exposing `IN_NONBLOCK` and `IN_CLOEXEC`. The returned fd is `read(2)`-able for `struct inotify_event` records, poll-able via `POLLIN`, and is the substrate for subsequent `inotify_add_watch(2)` / `inotify_rm_watch(2)` calls. Each instance maintains a watch table indexed by `wd` (watch descriptor) plus a bounded event queue (per-instance `max_queued_events`, default `16384`). Critical for: file-change monitoring under fanotify-incompatible mount/perm regimes, container observability, build-system change-trigger, systemd path units, in-tree fsnotify-mark accounting.

This Tier-5 covers the userspace ABI of syscall 294 (the **syscall**, not the header — that's `uapi/headers/inotify.md`).

## Signature

```c
int inotify_init1(int flags);
```

Rust ABI shim:

```rust
pub fn sys_inotify_init1(flags: i32) -> isize;
```

Syscall number: **294**.

Legacy companion `inotify_init(2)` (syscall 253):
- Signature: `int inotify_init(void);`
- Behavior identical to `inotify_init1(0)`.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `flags` | `i32` | IN | bitmask: `IN_NONBLOCK | IN_CLOEXEC` |

## Return

- **Success**: non-negative file descriptor.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `flags` contains an unknown bit |
| `EMFILE` | per-process fd table full (`RLIMIT_NOFILE`) or per-user inotify-instance limit hit (`fs.inotify.max_user_instances`) |
| `ENFILE` | system-wide fd table full |
| `ENOMEM` | kernel cannot allocate `fsnotify_group` |

## ABI surface

`IN_*` creation flags:

| Constant | Value | Purpose |
|---|---|---|
| `IN_CLOEXEC` | `O_CLOEXEC` (`0x80000`) | close-on-exec |
| `IN_NONBLOCK` | `O_NONBLOCK` (`0x800`) | non-blocking read |

Per-instance defaults (sysctl-tunable):
- `fs.inotify.max_user_instances` — per-uid instance cap (default `128`).
- `fs.inotify.max_user_watches` — per-uid watch cap across all instances (default `8192`/`65536`/`524288` depending on RAM).
- `fs.inotify.max_queued_events` — per-instance event-queue depth (default `16384`).

`struct inotify_event` (read frame, variable-length):

```c
struct inotify_event {
    __s32 wd;
    __u32 mask;
    __u32 cookie;
    __u32 len;
    char  name[];
};
```

## Compatibility contract

REQ-1: `flags` validation:
- Subset of `{IN_NONBLOCK, IN_CLOEXEC}`; else `-EINVAL`.

REQ-2: Returned fd:
- `O_RDONLY`-equivalent (writes return `-EBADF` from fop dispatch).
- `O_CLOEXEC` set iff `IN_CLOEXEC` flag.
- `O_NONBLOCK` set iff `IN_NONBLOCK` flag.
- Poll-able: `POLLIN ⟺ event queue non-empty`.
- `fcntl(F_GETFL)` / `F_SETFL` toggles `O_NONBLOCK` post-create.

REQ-3: Per-uid instance accounting:
- `inc_ucount(UCOUNT_INOTIFY_INSTANCES)`; failure ⟹ `-EMFILE`.
- Decremented in `release`.

REQ-4: Initial state:
- `fsnotify_group->inotify_data.queue` empty.
- `inotify_data.watches` IDR empty (no watches).
- `inotify_data.ucounts` link to caller's `user_namespace`.

REQ-5: anon_inode:
- Inode is anon_inode `inotify`.
- Visible in `/proc/<pid>/fd/<N>` as `anon_inode:inotify`.
- `/proc/<pid>/fdinfo/<N>` exposes `inotify wd:%x ino:%lx sdev:%x mask:%x ...` per watch.

REQ-6: Per-process fd accounting:
- Consumes one `RLIMIT_NOFILE` slot.

REQ-7: Lifetime:
- `close(2)` triggers `inotify_release`: drain queue, detach all marks, release group.
- `dup(2)` / `fork(2)`: shared `struct file`; both holders see same queue.
- `execve(2)`: closed iff `O_CLOEXEC`.

REQ-8: Read / poll semantics covered in `uapi/headers/inotify.md`; this Tier-5 only addresses creation.

REQ-9: Legacy `inotify_init()` ≡ `inotify_init1(0)`.

REQ-10: User-namespace scope:
- `ucounts` rooted at `current_user_ns()`; cap independently across namespaces.

REQ-11: Q-overflow:
- If queue exceeds `max_queued_events`, an `IN_Q_OVERFLOW` event (`wd == -1`) is enqueued and subsequent events are dropped until drained.

## Acceptance Criteria

- [ ] AC-1: `inotify_init1(0)` returns fd; watch table empty.
- [ ] AC-2: `inotify_init1(IN_CLOEXEC)`: `fcntl(fd, F_GETFD)` has `FD_CLOEXEC`.
- [ ] AC-3: `inotify_init1(IN_NONBLOCK)`: `fcntl(fd, F_GETFL)` has `O_NONBLOCK`.
- [ ] AC-4: `inotify_init1(IN_CLOEXEC | IN_NONBLOCK)`: both reflected.
- [ ] AC-5: `inotify_init1(0x4)` (unknown bit) → `-EINVAL`.
- [ ] AC-6: `inotify_init1(-1)` → `-EINVAL`.
- [ ] AC-7: `RLIMIT_NOFILE` exhausted → `-EMFILE`.
- [ ] AC-8: `fs.inotify.max_user_instances` exceeded → `-EMFILE`.
- [ ] AC-9: Legacy `inotify_init()` (syscall 253) ≡ `inotify_init1(0)`.
- [ ] AC-10: After `execve` with `IN_CLOEXEC`: fd absent.
- [ ] AC-11: Created fd shows as `anon_inode:inotify` in `/proc/<pid>/fd/`.
- [ ] AC-12: `read(fd, buf, sizeof(struct inotify_event))` on empty queue with `IN_NONBLOCK` → `-EAGAIN`.
- [ ] AC-13: Group structure pre-allocated before fd is installed (no torn install).
- [ ] AC-14: `close(fd)` decrements `UCOUNT_INOTIFY_INSTANCES`.

## Architecture

Rookery surface in `kernel/notify/inotify/syscall.rs`:

```rust
bitflags! {
    pub struct InotifyInitFlags: i32 {
        const CLOEXEC  = 0x80000;
        const NONBLOCK = 0x800;
    }
}

pub struct InotifyGroup {
    pub queue:       SpinLockedDeque<InotifyEvent>,
    pub watches:     RwLockedIdr<InotifyMark>,
    pub max_events:  u32,
    pub ucounts:     UCountsRef,
    pub wq:          WaitQueue,
    pub fanotify_id: u64,
}
```

`Inotify::inotify_init1(flags) -> isize`:
1. let f = InotifyInitFlags::from_bits(flags).ok_or(-EINVAL)?;
2. let ucounts = inc_ucount(UCOUNT_INOTIFY_INSTANCES).ok_or(-EMFILE)?;
3. let group = Arc::new(InotifyGroup {
     - queue: SpinLockedDeque::new(),
     - watches: RwLockedIdr::new(),
     - max_events: sysctl_max_queued_events(),
     - ucounts,
     - wq: WaitQueue::new(),
     - fanotify_id: next_group_id(),
   });
4. let open_flags = O_RDONLY
     | if f.contains(InotifyInitFlags::CLOEXEC) { O_CLOEXEC } else { 0 }
     | if f.contains(InotifyInitFlags::NONBLOCK) { O_NONBLOCK } else { 0 };
5. anon_inode_getfd("inotify", &INOTIFY_FOPS, group, open_flags).map(|fd| fd as isize)

`Inotify::inotify_init_legacy() -> isize`:
1. Inotify::inotify_init1(0).

`INOTIFY_FOPS`:
1. read     ⟹ Inotify::read (per uapi/headers/inotify.md).
2. poll     ⟹ Inotify::poll.
3. unlocked_ioctl ⟹ FIONREAD only.
4. release  ⟹ drain queue; detach marks; dec ucounts; free group.
5. show_fdinfo ⟹ enumerate watches as `inotify wd:%x ino:%lx sdev:%x mask:%x ignored_mask:0`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_subset` | INVARIANT | `flags & !ALL_FLAGS != 0 ⟹ -EINVAL` |
| `cloexec_propagates` | INVARIANT | `IN_CLOEXEC ⟹ FD_CLOEXEC set` |
| `nonblock_propagates` | INVARIANT | `IN_NONBLOCK ⟹ O_NONBLOCK set` |
| `ucount_balanced` | INVARIANT | inc on init, dec on release |
| `fd_install_atomic` | INVARIANT | fd visible iff group fully initialized |

### Layer 2: TLA+

`uapi/inotify_init1.tla`:
- States: `validating`, `accounting_ucount`, `allocating_group`, `installing_fd`, `returned`, `failed`.
- Properties:
  - `safety_ucount_paired_release` — every successful init must be matched by a release that decrements `UCOUNT_INOTIFY_INSTANCES`.
  - `safety_no_partial_install` — fd visible iff group fully initialized.
  - `liveness_terminate` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `inotify_init1` post(ok): `0 ≤ ret` ∧ group queue empty | `Inotify::inotify_init1` |
| `inotify_init1` post: group.ucounts increments by 1 | `Inotify::inotify_init1` |
| `inotify_init_legacy` post: equivalent to inotify_init1(0) | `Inotify::inotify_init_legacy` |

### Layer 4: Verus/Creusot functional

Per-`inotify(7)`/`inotify_init1(2)` man pages and `Documentation/filesystems/inotify.rst` semantic equivalence. Per-LTP `testcases/kernel/syscalls/inotify_init1/*.c` and glibc `sysdeps/unix/sysv/linux/inotify_init1.c` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

inotify_init1 reinforcement:

- **CLOEXEC default under restrictive policy** — defense against per-exec leak of inotify-group fd to setuid child.
- **UCount-based per-uid instance cap** — defense against per-user inotify-instance flood.
- **Queue depth capped by sysctl** — defense against per-group event-flood eating kernel memory.
- **anon_inode source isolation** — defense against per-namespace cross-leak.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF inherited** — the syscall takes only scalars; no user pointers. UDEREF discipline applies at the generic syscall entry path; the kernel never dereferences a user pointer in `inotify_init1` body.
- **PAX_RANDKSTACK on inotify_init1 entry** — randomize kernel stack at every fsnotify-group creation; `inotify` instances are durable observability primitives often opened from network-exposed daemons, making layout entropy at creation time worth the cost.
- **ucount + max_user_watches strict enforcement** — `fs.inotify.max_user_instances` and `fs.inotify.max_user_watches` are dual-counted under `UCOUNT_INOTIFY_*`; refuse creation past either cap with `-EMFILE`. Mirrors grsec doctrine that all userspace-creatable kernel objects must carry a per-uid quota distinct from the global cap, and that the per-uid cap must be tunable per-user-namespace.
- **GRKERNSEC_HIDESYM on `/proc/<pid>/fdinfo/<N>` watch entries** — fdinfo leaks `ino`, `sdev`, and `mask` per watch; under `kernel.kptr_restrict ≥ 2`, fdinfo readability restricted to `CAP_SYS_ADMIN` or same-uid + same-pidns observers. Without this, an unprivileged observer can fingerprint another process's watched paths even across mount-namespace boundaries.
- **GRKERNSEC_HARDEN_USERFAULTFD-style opt-in for inotify-on-procfs** — inotify on `/proc/<pid>/*` exposes signal-state side channels; under hardened policy, refuse `inotify_add_watch` on procfs entries unless `CAP_SYS_PTRACE` or `CAP_SYS_ADMIN`. The init1 path notes the restriction; the watch path enforces it.
- **GRKERNSEC_CHROOT_FINDTASK-style namespace fence** — when chroot is active and `grsec_chroot_findtask` is set, refuse `inotify_init1` from inside the chroot unless the policy explicitly permits it; chroot escape via inotify-events on outside paths is a documented vector.
- **Audit log on every `inotify_init1` from setuid-history task** — task with `dumpable == 0` creating an inotify group is logged at `LOGLEVEL_INFO` for forensic correlation; inotify groups can be inherited across exec to monitor post-exec mounts.
- **CAP_LINUX_IMMUTABLE not required, but observed** — inotify groups themselves are not immutable, but the marks they hold may be on `S_IMMUTABLE` inodes; the audit doctrine logs that combination.
- **Per-user-namespace ceiling tunable via /proc/sys** — even when the global cap is large, per-namespace caps must be settable to small values for untrusted containers; grsec doctrine requires sysctl knobs to honor `net.core.*` style ns-scoping.
- **No CAP_SYS_ADMIN for IN_NONBLOCK / IN_CLOEXEC** — baseline primitive; capabilities not required for init1, but the watch path may require them for procfs / sysfs / cgroupfs watches under hardened policy.
- **inotify event-queue rate-limit** — under hardened policy, refuse to enqueue more than `N` events per millisecond from a single mark; replaces the current behavior of silently dropping with `IN_Q_OVERFLOW` and adds an audit-log entry for the rate-limit trip.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `uapi/headers/inotify.md` Tier-5 — header-level read frame, `struct inotify_event`, poll semantics
- `inotify_add_watch.md` Tier-5 — watch installation
- `inotify_rm_watch.md` Tier-5 — watch removal
- `fs/notify/inotify.md` Tier-3 — `fsnotify_group` / `inotify_mark` lifecycle
- Implementation code
