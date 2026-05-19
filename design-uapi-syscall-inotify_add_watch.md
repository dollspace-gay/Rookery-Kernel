---
title: "Tier-5 syscall: inotify_add_watch(2) — syscall 254"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`inotify_add_watch(2)` attaches (or updates) an `inotify_mark` on the inode identified by `pathname` against the inotify instance referred to by `fd`. Returns a non-negative watch descriptor (`wd`) — a per-group monotonically allocated integer that subsequent events carry. The `mask` argument is a bitmask of `IN_*` event bits plus modifier bits (`IN_MASK_ADD`, `IN_MASK_CREATE`, `IN_ONESHOT`, `IN_ONLYDIR`, `IN_EXCL_UNLINK`, `IN_DONT_FOLLOW`). If a watch on the same inode already exists in the group: default behavior is replace; `IN_MASK_ADD` merges; `IN_MASK_CREATE` fails with `-EEXIST`. Critical for: build systems (watchman, chokidar), systemd path units, fileindexers, container runtime mount monitoring, log rotators.

This Tier-5 covers the userspace ABI of syscall 254.

### Acceptance Criteria

- [ ] AC-1: `inotify_add_watch(fd, "/tmp/x", IN_MODIFY)` returns `wd >= 1`.
- [ ] AC-2: `IN_MASK_ADD`: second call merges mask, returns same `wd`.
- [ ] AC-3: `IN_MASK_CREATE` on existing watch → `-EEXIST`.
- [ ] AC-4: `mask = 0` → `-EINVAL`.
- [ ] AC-5: `mask = IN_MASK_ADD` (no event bit) → `-EINVAL`.
- [ ] AC-6: `mask = IN_MASK_ADD | IN_MASK_CREATE | IN_MODIFY` → `-EINVAL`.
- [ ] AC-7: `mask = 0x00800000` (reserved bit) → `-EINVAL`.
- [ ] AC-8: Bad fd → `-EBADF`.
- [ ] AC-9: Bad fd type (e.g., regular file) → `-EINVAL`.
- [ ] AC-10: Path not found → `-ENOENT`.
- [ ] AC-11: `IN_ONLYDIR` on regular file → `-ENOTDIR`.
- [ ] AC-12: Read-denied path → `-EACCES`.
- [ ] AC-13: `fs.inotify.max_user_watches` exceeded → `-ENOSPC`.
- [ ] AC-14: `IN_ONESHOT`: after first event, second event not delivered.
- [ ] AC-15: `IN_DONT_FOLLOW` on symlink: watch on symlink itself, not target.

### Architecture

Rookery surface in `kernel/notify/inotify/syscall.rs`:

```rust
bitflags! {
    pub struct InMask: u32 {
        const ACCESS         = 0x00000001;
        const MODIFY         = 0x00000002;
        /* ... full table omitted for brevity ... */
        const ALL_EVENTS     = 0x00000fff;
        const ONLYDIR        = 0x01000000;
        const DONT_FOLLOW    = 0x02000000;
        const EXCL_UNLINK    = 0x04000000;
        const MASK_CREATE    = 0x10000000;
        const MASK_ADD       = 0x20000000;
        const ONESHOT        = 0x80000000;
    }
}

pub struct InotifyMark {
    pub fsn_mark: FsnotifyMark,
    pub wd: i32,
}
```

`Inotify::inotify_add_watch(fd, path_user, mask) -> isize`:
1. let f = InMask::from_bits(mask).ok_or(-EINVAL)?;
2. require f.intersects(InMask::ALL_EVENTS); else -EINVAL.
3. if f.contains(InMask::MASK_CREATE | InMask::MASK_ADD): return -EINVAL.
4. let file = fdget(fd).ok_or(-EBADF)?;
5. let group = file.private_data.downcast::<InotifyGroup>().ok_or(-EINVAL)?;
6. let path = copy_path_from_user(path_user, PATH_MAX)?;
7. let lookup_flags = if f.contains(InMask::DONT_FOLLOW) { 0 } else { LOOKUP_FOLLOW };
8. let lookup_flags = lookup_flags | if f.contains(InMask::ONLYDIR) { LOOKUP_DIRECTORY } else { 0 };
9. let inode = user_path_at(AT_FDCWD, &path, lookup_flags)?;
10. inode_permission(&inode, MAY_READ).map_err(|_| -EACCES)?;
11. let mut marks = group.watches.write();
12. if let Some(existing) = marks.find_by_inode(&inode) {
      - if f.contains(InMask::MASK_CREATE): return -EEXIST;
      - if f.contains(InMask::MASK_ADD): existing.fsn_mark.mask |= (f.bits() & InMask::ALL_EVENTS.bits()) | modifier_bits;
      - else: existing.fsn_mark.mask = f.bits();
      - return existing.wd as isize;
   }
13. inc_ucount(UCOUNT_INOTIFY_WATCHES).ok_or(-ENOSPC)?;
14. let mark = InotifyMark { fsn_mark: FsnotifyMark::new(group, inode, f.bits()), wd: marks.alloc_wd()? };
15. fsnotify_add_inode_mark(&mark.fsn_mark, &inode, /*allow_dups=*/0)?;
16. marks.install(mark.wd, mark);
17. return mark.wd as isize;

### Out of Scope

- `inotify_init1.md` Tier-5 — instance creation
- `inotify_rm_watch.md` Tier-5 — watch removal
- `uapi/headers/inotify.md` Tier-5 — read frame, poll, event semantics
- `fs/notify/inotify.md` Tier-3 — fsnotify_mark / inode mark mechanics
- Implementation code

### signature

```c
int inotify_add_watch(int fd, const char *pathname, uint32_t mask);
```

Rust ABI shim:

```rust
pub fn sys_inotify_add_watch(fd: i32, pathname: *const u8, mask: u32) -> isize;
```

Syscall number: **254**.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `fd` | `i32` | IN | inotify instance from `inotify_init1(2)` |
| `pathname` | `*const u8` | IN (user) | NUL-terminated path; `PATH_MAX` capped |
| `mask` | `u32` | IN | bitmask: events + modifiers |

### return

- **Success**: non-negative watch descriptor (`wd >= 1`).
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EACCES` | read permission on target inode denied |
| `EBADF` | `fd` not an inotify fd |
| `EEXIST` | `IN_MASK_CREATE` set and watch already exists |
| `EFAULT` | `pathname` not accessible |
| `EINVAL` | `mask` has no event bit set, or unknown bits, or both `IN_MASK_CREATE` and `IN_MASK_ADD` |
| `ENAMETOOLONG` | `pathname` > `PATH_MAX` |
| `ENOENT` | `pathname` does not resolve |
| `ENOMEM` | watch alloc failed |
| `ENOSPC` | per-uid `fs.inotify.max_user_watches` cap hit |
| `ENOTDIR` | `IN_ONLYDIR` set but target not a directory |

### abi surface

`IN_*` event bits (mask):

| Constant | Value | Trigger |
|---|---|---|
| `IN_ACCESS` | `0x00000001` | file accessed |
| `IN_MODIFY` | `0x00000002` | file modified |
| `IN_ATTRIB` | `0x00000004` | metadata changed |
| `IN_CLOSE_WRITE` | `0x00000008` | writable fd closed |
| `IN_CLOSE_NOWRITE` | `0x00000010` | read-only fd closed |
| `IN_OPEN` | `0x00000020` | file opened |
| `IN_MOVED_FROM` | `0x00000040` | file moved out of watched dir |
| `IN_MOVED_TO` | `0x00000080` | file moved into watched dir |
| `IN_CREATE` | `0x00000100` | child created |
| `IN_DELETE` | `0x00000200` | child deleted |
| `IN_DELETE_SELF` | `0x00000400` | watched inode deleted |
| `IN_MOVE_SELF` | `0x00000800` | watched inode moved |
| `IN_ALL_EVENTS` | `0x00000fff` | mask of all above |

Modifier bits:

| Constant | Value | Effect |
|---|---|---|
| `IN_ONLYDIR` | `0x01000000` | fail if not directory |
| `IN_DONT_FOLLOW` | `0x02000000` | do not follow final symlink |
| `IN_EXCL_UNLINK` | `0x04000000` | suppress events after unlink |
| `IN_MASK_CREATE` | `0x10000000` | fail if mark exists |
| `IN_MASK_ADD` | `0x20000000` | OR into existing mask |
| `IN_ONESHOT` | `0x80000000` | one-shot: detach after first event |

Per-uid caps (sysctl):
- `fs.inotify.max_user_watches` — total watches across all instances of the uid.

### compatibility contract

REQ-1: `fd` validation:
- Must be an inotify-fd (file_operations match); else `-EBADF`.

REQ-2: `pathname` validation:
- `strncpy_from_user` with `PATH_MAX` cap.
- Empty pathname ⟹ `-ENOENT`.

REQ-3: `mask` validation:
- Must contain at least one event bit from `IN_ALL_EVENTS`; else `-EINVAL`.
- Must not contain unknown bits; else `-EINVAL`.
- `IN_MASK_CREATE | IN_MASK_ADD` simultaneously ⟹ `-EINVAL`.

REQ-4: Path resolution:
- `user_path_at(AT_FDCWD, pathname, lookup_flags, &path)`.
- `lookup_flags = LOOKUP_FOLLOW` unless `IN_DONT_FOLLOW`.
- `IN_ONLYDIR` ⟹ require directory inode; else `-ENOTDIR`.
- Read-permission check via `inode_permission(MAY_READ)`; failure ⟹ `-EACCES`.

REQ-5: Watch installation:
- Look up existing mark on (group, inode):
  - If exists ∧ `IN_MASK_CREATE`: `-EEXIST`.
  - If exists ∧ `IN_MASK_ADD`: `new_mask = old_mask | (mask & ~IN_MASK_ADD)`.
  - If exists otherwise: `new_mask = mask`.
  - Else allocate new mark:
    - `inc_ucount(UCOUNT_INOTIFY_WATCHES)` enforces `max_user_watches`; failure ⟹ `-ENOSPC`.
    - `idr_alloc_cyclic` for new `wd`; failure ⟹ `-ENOSPC`.

REQ-6: Mark attachment:
- `fsnotify_add_inode_mark(mark, inode, allow_dups=0)`.

REQ-7: Returned wd:
- Monotonically positive integer (`wd >= 1`); zero reserved.

REQ-8: `IN_ONESHOT`:
- Mark detached after first delivered event; future events on inode silent.

REQ-9: `IN_EXCL_UNLINK`:
- After unlinking watched child, no further events from that inode are delivered to this watch.

REQ-10: User-namespace scope:
- `ucounts` rooted at instance's creation user_ns.

REQ-11: `wd` lifetime:
- Valid until `inotify_rm_watch(2)` or inode removal (`IN_IGNORED` event).

REQ-12: Special filesystems:
- `procfs`, `sysfs`, `cgroupfs`, `debugfs` — supported with caveats (some inodes static; events sparse).
- `tmpfs`, `overlayfs` — fully supported.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mask_has_event` | INVARIANT | (mask & ALL_EVENTS) == 0 ⟹ -EINVAL |
| `mask_create_xor_add` | INVARIANT | (CREATE ∧ ADD) ⟹ -EINVAL |
| `unknown_bits_rejected` | INVARIANT | mask & RESERVED ⟹ -EINVAL |
| `wd_monotonic` | INVARIANT | returned wd > 0 ∧ unique per group until rm |
| `ucount_balanced` | INVARIANT | inc on new mark, dec on rm/release |
| `path_copy_bounded` | INVARIANT | path copy ≤ PATH_MAX |

### Layer 2: TLA+

`uapi/inotify_add_watch.tla`:
- States: `validating_mask`, `fdget`, `path_lookup`, `perm_check`, `mark_lookup`, `mark_install`, `returned`, `failed`.
- Properties:
  - `safety_no_double_install` — MASK_CREATE forbids replace.
  - `safety_mask_merge` — MASK_ADD: new ⊇ old.
  - `safety_oneshot_self_detach` — ONESHOT mark removed after first delivery.
  - `liveness_terminate` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `inotify_add_watch` post(ok): wd ≥ 1 ∧ mark installed | `Inotify::inotify_add_watch` |
| MASK_ADD merges, default replaces | `Inotify::update_or_install` |
| ucount inc paired with rm/release dec | `Inotify::rm_watch`, `Inotify::release` |

### Layer 4: Verus/Creusot functional

Per-`inotify(7)` and `inotify_add_watch(2)` man pages, `Documentation/filesystems/inotify.rst` semantic equivalence. Per-LTP `testcases/kernel/syscalls/inotify_add_watch/*.c` round-trip.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

inotify_add_watch reinforcement:

- **Per-uid `max_user_watches` strict cap** — defense against per-user watch flood eating kernel memory.
- **MAY_READ permission check** — defense against per-unprivileged-observer fingerprinting.
- **`IN_EXCL_UNLINK` post-unlink suppression** — defense against per-stale-inode event leak.
- **`IN_ONESHOT` auto-detach** — defense against per-orphaned-mark resource leak.
- **Path-copy `PATH_MAX` ceiling** — defense against per-unbounded copy from user.

### grsecurity/pax-style reinforcement

- **PaX UDEREF on `pathname` user buffer** — the kernel never touches the pointer directly; `strncpy_from_user` with `PATH_MAX` bound and `access_ok` precheck. Under UDEREF, the user-space mapping is shadow-checked before each byte; SMAP/SMEP must be active on the entry. Refuse if `pathname` lies in a non-readable user mapping.
- **GRKERNSEC_CHROOT_FINDTASK-style fence** — when caller is inside a chroot and the resolved `path` escapes the chroot (e.g., via `..` resolution prior to the chroot root binding), refuse with `-EACCES`. Inotify on outside-chroot paths is a documented escape vector for compromised chroot daemons.
- **GRKERNSEC_HIDESYM on fdinfo per-watch records** — `/proc/<pid>/fdinfo/<N>` per-watch entries (`ino:%lx`, `sdev:%x`, `mask:%x`) leak inode numbers and device IDs across users; under `kernel.kptr_restrict ≥ 2`, fdinfo readability restricted to same-uid + same-pidns observers or `CAP_SYS_ADMIN`.
- **ucount + max_user_watches dual-counted** — `fs.inotify.max_user_watches` enforced by `UCOUNT_INOTIFY_WATCHES`; refuse past either the per-uid or per-namespace cap with `-ENOSPC`. Mirrors the grsec doctrine that user-creatable kernel objects must carry per-uid quotas distinct from global caps.
- **GRKERNSEC_HARDEN_USERFAULTFD-style opt-in for procfs/sysfs watches** — refuse `inotify_add_watch` on `/proc/<pid>/*` and `/sys/*` paths unless caller holds `CAP_SYS_PTRACE` or `CAP_SYS_ADMIN`. Procfs watches are a known side channel for signal-state inference and for fingerprinting unrelated processes.
- **PAX_RANDKSTACK on every watch add** — randomize kernel stack on each install; `inotify_add_watch` is the high-frequency entry point and the mark-allocator interaction with the idr is a useful target for kstack-layout inference.
- **Audit log on every watch from setuid-history task** — `dumpable == 0` task installing a watch is logged at `LOGLEVEL_INFO`; watches established before privilege drop and surviving exec are a common forensic artifact.
- **IN_ONLYDIR + LOOKUP_DIRECTORY hard-enforced** — refuse symlinks-pointing-to-directories under `IN_ONLYDIR`; the lookup must yield a true directory, not a symlink that resolves to one. Defense against TOCTOU symlink-swap into a directory mid-call.
- **Refuse watches on /sys/kernel/debug, /sys/firmware, and /dev/kmsg under hardened policy** — those paths leak kernel-internal state to userspace observers; under `kernel.dmesg_restrict = 1`, watches on `/dev/kmsg` require `CAP_SYS_ADMIN`.
- **mask-bit RESERVED-zero strict** — bits not in the documented mask are reserved; refuse with `-EINVAL` rather than silently accept. Mirrors grsec's strict-validation doctrine that future bit assignments must not be aliased by old binaries.
- **inotify-on-symlink chained-resolution audit** — when `IN_DONT_FOLLOW` is absent and the path resolution traverses N symlinks, log N for N > 4; chained symlink attacks against build-system watchers are a documented vector.

