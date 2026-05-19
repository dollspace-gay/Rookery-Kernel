# Tier-5 UAPI: include/uapi/linux/fanotify.h — fanotify(7) ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/fanotify.h (~274 lines)
  - fs/notify/fanotify/fanotify_user.c
  - fs/notify/fanotify/fanotify.c
  - fs/notify/fanotify/fanotify.h
  - include/linux/fanotify.h
-->

## Summary

`fanotify(7)` is the filesystem-scope notification API that supersedes inotify for kernel-mediated access control. A per-group fd is created by `fanotify_init(flags, event_f_flags)`, watches (marks) are added with `fanotify_mark(fd, flags, mask, dirfd, pathname)`, and events stream out via `read(2)` as a variable-length `struct fanotify_event_metadata { __u32 event_len; __u8 vers; __u8 reserved; __u16 metadata_len; __aligned_u64 mask; __s32 fd; __s32 pid; }` optionally followed by `struct fanotify_event_info_*` records (FID / DFID / DFID_NAME / PIDFD / ERROR / RANGE / MNT). Permission-class events (`FAN_ACCESS_PERM`, `FAN_OPEN_PERM`, `FAN_OPEN_EXEC_PERM`) block the originating syscall until userspace writes back a `struct fanotify_response { __s32 fd; __u32 response; }` with `FAN_ALLOW` or `FAN_DENY` (errno-bits-encoded for non-EPERM denial). Per-group flag classes: notification-only (`FAN_CLASS_NOTIF`), content scan (`FAN_CLASS_CONTENT`), pre-content scan (`FAN_CLASS_PRE_CONTENT`); per-flag report toggles (`FAN_REPORT_FID`/`DIR_FID`/`NAME`/`PIDFD`/`TID`/`TARGET_FID`); per-flag queue policy (`FAN_UNLIMITED_QUEUE`/`MARKS`, `FAN_ENABLE_AUDIT`). Mark scopes: inode, mount, filesystem, mount-namespace. Critical for: antivirus / EDR, container image scanning, audit, hierarchical-storage management, FAN_FS_ERROR fs-error telemetry.

This Tier-5 covers `include/uapi/linux/fanotify.h` (~274 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `sys_fanotify_init` | per-group fd alloc + class/flag config | `Fanotify::sys_init` |
| `sys_fanotify_mark` | per-mark add/remove/flush | `Fanotify::sys_mark` |
| `struct fanotify_event_metadata` | per-event userspace record | `uapi::fanotify::EventMetadata` |
| `struct fanotify_event_info_header` | per-info variable record header | `uapi::fanotify::EventInfoHeader` |
| `struct fanotify_event_info_fid` | per-FID/DFID/DFID_NAME record | `uapi::fanotify::EventInfoFid` |
| `struct fanotify_event_info_pidfd` | per-PIDFD record | `uapi::fanotify::EventInfoPidfd` |
| `struct fanotify_event_info_error` | per-FAN_FS_ERROR record | `uapi::fanotify::EventInfoError` |
| `struct fanotify_event_info_range` | per-FAN_PRE_ACCESS range record | `uapi::fanotify::EventInfoRange` |
| `struct fanotify_event_info_mnt` | per-FAN_MNT_* mount record | `uapi::fanotify::EventInfoMnt` |
| `struct fanotify_response` | per-permission ALLOW/DENY response | `uapi::fanotify::Response` |
| `struct fanotify_response_info_header` | per-response info header | `uapi::fanotify::ResponseInfoHeader` |
| `struct fanotify_response_info_audit_rule` | per-response audit-rule record | `uapi::fanotify::ResponseInfoAuditRule` |
| `FANOTIFY_METADATA_VERSION` (3) | per-ABI version constant | shared |
| `FAN_EVENT_NEXT` / `FAN_EVENT_OK` | per-iter helpers | shared inline |

## ABI surface (constants + structs)

### Event mask bits (used in mark mask and event metadata)

| Bit | Constant | Class | Semantics |
|---|---|---|---|
| 0x00000001 | `FAN_ACCESS` | notif | file read |
| 0x00000002 | `FAN_MODIFY` | notif | file write |
| 0x00000004 | `FAN_ATTRIB` | notif | metadata change |
| 0x00000008 | `FAN_CLOSE_WRITE` | notif | writable fd closed |
| 0x00000010 | `FAN_CLOSE_NOWRITE` | notif | unwritable fd closed |
| 0x00000020 | `FAN_OPEN` | notif | file opened |
| 0x00000040 | `FAN_MOVED_FROM` | notif | dirent moved away |
| 0x00000080 | `FAN_MOVED_TO` | notif | dirent moved in |
| 0x00000100 | `FAN_CREATE` | notif | dirent created |
| 0x00000200 | `FAN_DELETE` | notif | dirent removed |
| 0x00000400 | `FAN_DELETE_SELF` | notif | watched object unlinked |
| 0x00000800 | `FAN_MOVE_SELF` | notif | watched object renamed |
| 0x00001000 | `FAN_OPEN_EXEC` | notif | file opened for exec |
| 0x00004000 | `FAN_Q_OVERFLOW` | notif | per-group queue overflowed |
| 0x00008000 | `FAN_FS_ERROR` | notif | filesystem error reported |
| 0x00010000 | `FAN_OPEN_PERM` | perm | open in permission check |
| 0x00020000 | `FAN_ACCESS_PERM` | perm | access in permission check |
| 0x00040000 | `FAN_OPEN_EXEC_PERM` | perm | open-for-exec in permission check |
| 0x00100000 | `FAN_PRE_ACCESS` | perm | pre-content read range hook |
| 0x01000000 | `FAN_MNT_ATTACH` | notif | mount attached (FAN_REPORT_MNT) |
| 0x02000000 | `FAN_MNT_DETACH` | notif | mount detached (FAN_REPORT_MNT) |
| 0x08000000 | `FAN_EVENT_ON_CHILD` | mark mod | mark also delivers events from immediate children |
| 0x10000000 | `FAN_RENAME` | notif | rename (with `FAN_REPORT_*FID*`) |
| 0x40000000 | `FAN_ONDIR` | notif | event subject is a directory |

```
#define FAN_CLOSE  (FAN_CLOSE_WRITE | FAN_CLOSE_NOWRITE)
#define FAN_MOVE   (FAN_MOVED_FROM  | FAN_MOVED_TO)
```

### `fanotify_init(flags, event_f_flags)` — `flags`

| Bit | Constant | Semantics |
|---|---|---|
| 0x00000001 | `FAN_CLOEXEC` | fd closed on exec |
| 0x00000002 | `FAN_NONBLOCK` | read returns `EAGAIN` if empty |
| 0x00000000 | `FAN_CLASS_NOTIF` | notification only (no perm events) |
| 0x00000004 | `FAN_CLASS_CONTENT` | post-content scan (perm events on open/access) |
| 0x00000008 | `FAN_CLASS_PRE_CONTENT` | pre-content scan (HSM-style stage-in) |
| 0x00000010 | `FAN_UNLIMITED_QUEUE` | bypass per-group queue cap (CAP_SYS_ADMIN) |
| 0x00000020 | `FAN_UNLIMITED_MARKS` | bypass per-group mark cap (CAP_SYS_ADMIN) |
| 0x00000040 | `FAN_ENABLE_AUDIT` | record permission decisions via audit (CAP_AUDIT_WRITE) |
| 0x00000080 | `FAN_REPORT_PIDFD` | emit `event_info_pidfd` for events |
| 0x00000100 | `FAN_REPORT_TID` | `pid` field is thread id, not tgid |
| 0x00000200 | `FAN_REPORT_FID` | emit `event_info_fid` (FID) for events |
| 0x00000400 | `FAN_REPORT_DIR_FID` | emit `event_info_fid` (DFID) for dirent events |
| 0x00000800 | `FAN_REPORT_NAME` | emit DFID_NAME (requires FAN_REPORT_DIR_FID) |
| 0x00001000 | `FAN_REPORT_TARGET_FID` | emit rename target FID (requires all FID flags) |
| 0x00002000 | `FAN_REPORT_FD_ERROR` | report fd errors instead of dropping events |
| 0x00004000 | `FAN_REPORT_MNT` | report mount-namespace mount attach/detach |

Convenience:
```
#define FAN_REPORT_DFID_NAME         (FAN_REPORT_DIR_FID | FAN_REPORT_NAME)
#define FAN_REPORT_DFID_NAME_TARGET  (FAN_REPORT_DFID_NAME | FAN_REPORT_FID | FAN_REPORT_TARGET_FID)
```

`event_f_flags` are the `O_*` flags applied to the per-event fd handed to userspace (e.g. `O_RDONLY | O_CLOEXEC | O_LARGEFILE`).

### `fanotify_mark(fd, flags, mask, dirfd, pathname)` — `flags`

Action bits (mutually exclusive among `ADD` / `REMOVE` / `FLUSH`):

| Bit | Constant | Semantics |
|---|---|---|
| 0x00000001 | `FAN_MARK_ADD` | add bits in `mask` to mark |
| 0x00000002 | `FAN_MARK_REMOVE` | remove bits in `mask` from mark |
| 0x00000080 | `FAN_MARK_FLUSH` | flush all marks of a given scope on the group |

Modifier bits:

| Bit | Constant | Semantics |
|---|---|---|
| 0x00000004 | `FAN_MARK_DONT_FOLLOW` | do not follow symlink at leaf |
| 0x00000008 | `FAN_MARK_ONLYDIR` | fail with `ENOTDIR` if not a directory |
| 0x00000020 | `FAN_MARK_IGNORED_MASK` | bits in `mask` define events to *not* report (legacy) |
| 0x00000040 | `FAN_MARK_IGNORED_SURV_MODIFY` | ignored mask survives `MODIFY` events |
| 0x00000200 | `FAN_MARK_EVICTABLE` | mark may be reclaimed under memory pressure (inode marks) |
| 0x00000400 | `FAN_MARK_IGNORE` | new-style ignore (mutually exclusive with `FAN_MARK_IGNORED_MASK`) |

Scope bits (NOT bitwise-OR-able freely; only the documented compound `MNTNS` is composite):

| Value | Constant | Scope |
|---|---|---|
| 0x00000000 | `FAN_MARK_INODE` | inode-scope mark |
| 0x00000010 | `FAN_MARK_MOUNT` | mount-scope mark |
| 0x00000100 | `FAN_MARK_FILESYSTEM` | superblock-scope mark |
| 0x00000110 | `FAN_MARK_MNTNS` | mount-namespace-scope mark (= MOUNT \| FILESYSTEM bit pattern reserved) |

Convenience for non-inode marks (post-ABI-fixup):
```
#define FAN_MARK_IGNORE_SURV  (FAN_MARK_IGNORE | FAN_MARK_IGNORED_SURV_MODIFY)
```

### Struct `fanotify_event_metadata`

```
struct fanotify_event_metadata {
    __u32         event_len;     /* total record size: metadata + all info records */
    __u8          vers;          /* FANOTIFY_METADATA_VERSION (== 3) */
    __u8          reserved;      /* must be 0 */
    __u16         metadata_len;  /* sizeof(struct fanotify_event_metadata) */
    __aligned_u64 mask;          /* FAN_* event bits that triggered */
    __s32         fd;            /* event fd, FAN_NOFD (-1), or FAN_EPIDFD (-2) */
    __s32         pid;           /* tgid (or tid if FAN_REPORT_TID) */
};
#define FANOTIFY_METADATA_VERSION 3
#define FAN_EVENT_METADATA_LEN  (sizeof(struct fanotify_event_metadata))
```

Iteration helpers (inline-defined in UAPI):

```
#define FAN_EVENT_NEXT(meta, len) \
    ((len) -= (meta)->event_len, \
     (struct fanotify_event_metadata *)(((char *)(meta)) + (meta)->event_len))

#define FAN_EVENT_OK(meta, len) \
    ((long)(len) >= (long)FAN_EVENT_METADATA_LEN && \
     (long)(meta)->event_len >= (long)FAN_EVENT_METADATA_LEN && \
     (long)(meta)->event_len <= (long)(len))
```

### Variable info records

```
struct fanotify_event_info_header {
    __u8  info_type;   /* FAN_EVENT_INFO_TYPE_* */
    __u8  pad;
    __u16 len;         /* size of this info record including header */
};

struct fanotify_event_info_fid {
    struct fanotify_event_info_header hdr;
    __kernel_fsid_t fsid;
    unsigned char   handle[];   /* opaque struct file_handle for open_by_handle_at(2) */
};

struct fanotify_event_info_pidfd {
    struct fanotify_event_info_header hdr;
    __s32 pidfd;
};

struct fanotify_event_info_error {
    struct fanotify_event_info_header hdr;
    __s32 error;       /* errno */
    __u32 error_count; /* coalesced count of identical errors */
};

struct fanotify_event_info_range {
    struct fanotify_event_info_header hdr;
    __u32 pad;
    __u64 offset;
    __u64 count;
};

struct fanotify_event_info_mnt {
    struct fanotify_event_info_header hdr;
    __u64 mnt_id;
};
```

Info-type tag values:

| Value | Constant | Used for |
|---|---|---|
| 1 | `FAN_EVENT_INFO_TYPE_FID` | object FID (FAN_REPORT_FID) |
| 2 | `FAN_EVENT_INFO_TYPE_DFID_NAME` | parent DFID + child name |
| 3 | `FAN_EVENT_INFO_TYPE_DFID` | parent DFID |
| 4 | `FAN_EVENT_INFO_TYPE_PIDFD` | per-event pidfd |
| 5 | `FAN_EVENT_INFO_TYPE_ERROR` | per-FAN_FS_ERROR descriptor |
| 6 | `FAN_EVENT_INFO_TYPE_RANGE` | per-FAN_PRE_ACCESS byte range |
| 7 | `FAN_EVENT_INFO_TYPE_MNT` | per-FAN_MNT_* mount id |
| 10 | `FAN_EVENT_INFO_TYPE_OLD_DFID_NAME` | per-FAN_RENAME source |
| 12 | `FAN_EVENT_INFO_TYPE_NEW_DFID_NAME` | per-FAN_RENAME destination |

(Values 11 and 13 are reserved for the matching `OLD_DFID` / `NEW_DFID` variants.)

### Per-permission response

```
struct fanotify_response {
    __s32 fd;          /* must match the event fd */
    __u32 response;    /* FAN_ALLOW | FAN_DENY | FAN_AUDIT | FAN_INFO | FAN_DENY_ERRNO(err) */
};

#define FAN_ALLOW            0x01
#define FAN_DENY             0x02
#define FAN_AUDIT            0x10   /* record this decision via audit */
#define FAN_INFO             0x20   /* response is followed by response_info_* */

#define FAN_ERRNO_BITS       8
#define FAN_ERRNO_SHIFT      (32 - FAN_ERRNO_BITS)
#define FAN_ERRNO_MASK       ((1 << FAN_ERRNO_BITS) - 1)
#define FAN_DENY_ERRNO(err)  (FAN_DENY | (((__u32)(err)) & FAN_ERRNO_MASK) << FAN_ERRNO_SHIFT)

struct fanotify_response_info_header {
    __u8  type;        /* FAN_RESPONSE_INFO_* */
    __u8  pad;
    __u16 len;
};

struct fanotify_response_info_audit_rule {
    struct fanotify_response_info_header hdr;
    __u32 rule_number;
    __u32 subj_trust;
    __u32 obj_trust;
};

#define FAN_RESPONSE_INFO_NONE        0
#define FAN_RESPONSE_INFO_AUDIT_RULE  1
```

Sentinels:

```
#define FAN_NOFD    -1
#define FAN_NOPIDFD FAN_NOFD
#define FAN_EPIDFD  -2
```

### Deprecated aggregate constants (do not use for new code)

`FAN_ALL_CLASS_BITS`, `FAN_ALL_INIT_FLAGS`, `FAN_ALL_MARK_FLAGS`, `FAN_ALL_EVENTS`, `FAN_ALL_PERM_EVENTS`, `FAN_ALL_OUTGOING_EVENTS` — preserved as ABI-stable but explicitly marked "do not add new flags here" in upstream.

## Compatibility contract

REQ-1: `sys_fanotify_init(flags, event_f_flags)`:
- /* Permission gate */
- Require CAP_SYS_ADMIN unless `FAN_CLASS_NOTIF` ∧ no `FAN_UNLIMITED_*` ∧ no `FAN_ENABLE_AUDIT` ∧ a user-namespace-allowed subset (per `fanotify_user.c::fanotify_init`).
- /* Validate flags */
- exactly one of FAN_CLASS_NOTIF / CONTENT / PRE_CONTENT.
- if (flags & FAN_REPORT_NAME) ∧ !(flags & FAN_REPORT_DIR_FID): -EINVAL.
- if (flags & FAN_REPORT_TARGET_FID) ∧ ((flags & FAN_REPORT_DFID_NAME_TARGET) != FAN_REPORT_DFID_NAME_TARGET): -EINVAL.
- if (flags & FAN_UNLIMITED_QUEUE) ∧ !capable(CAP_SYS_ADMIN): -EPERM.
- if (flags & FAN_UNLIMITED_MARKS) ∧ !capable(CAP_SYS_ADMIN): -EPERM.
- if (flags & FAN_ENABLE_AUDIT) ∧ !capable(CAP_AUDIT_WRITE): -EPERM.
- /* event_f_flags */
- O_RDONLY | O_WRONLY | O_RDWR | O_LARGEFILE | O_CLOEXEC | O_APPEND | O_NONBLOCK | O_NOATIME accepted; others -EINVAL.
- Allocate group, anon-inode fd; return fd.

REQ-2: `sys_fanotify_mark(fd, flags, mask, dirfd, pathname)`:
- /* Validate action */
- action_bits = flags & (FAN_MARK_ADD | FAN_MARK_REMOVE | FAN_MARK_FLUSH).
- if popcount(action_bits) != 1: -EINVAL.
- /* Validate scope */
- scope_bits = flags & (FAN_MARK_INODE | FAN_MARK_MOUNT | FAN_MARK_FILESYSTEM | FAN_MARK_MNTNS).
- Exactly one of INODE / MOUNT / FILESYSTEM / MNTNS (INODE has value 0; choose by absence of others).
- /* Validate mask */
- /* No FAN_ONDIR alone */
- if mask & ~(supported_event_mask_for_group): -EINVAL.
- /* Per-IGNORE rules */
- if (flags & FAN_MARK_IGNORE) ∧ (flags & FAN_MARK_IGNORED_MASK): -EINVAL.
- if scope != INODE ∧ (flags & FAN_MARK_IGNORE) ∧ !(flags & FAN_MARK_IGNORED_SURV_MODIFY): -EINVAL.
- /* Resolve path */
- lookup_flags = LOOKUP_FOLLOW.
- if flags & FAN_MARK_DONT_FOLLOW: lookup_flags &= ~LOOKUP_FOLLOW.
- if flags & FAN_MARK_ONLYDIR: lookup_flags |= LOOKUP_DIRECTORY.
- /* Find/create mark */
- mark = fanotify_find_mark(group, target).
- match action_bits:
  - FAN_MARK_ADD: mark.mask |= mask; (or ignored_mask if FAN_MARK_IGNORE*).
  - FAN_MARK_REMOVE: mark.mask &= ~mask; if mark.mask == 0 ∧ mark.ignored_mask == 0: destroy.
  - FAN_MARK_FLUSH: destroy all marks of this scope in this group.
- return 0.

REQ-3: Per-event format on `read(fd)`:
- One or more concatenated records.
- Each record begins with `fanotify_event_metadata`; `event_len` covers metadata + all trailing info records.
- `vers == FANOTIFY_METADATA_VERSION`. `reserved == 0`. `metadata_len == sizeof(struct fanotify_event_metadata)`.
- `fd`:
  - For notification events without `FAN_REPORT_FID`: opened with `event_f_flags`.
  - With `FAN_REPORT_FID` (and no perm class): `fd == FAN_NOFD`; FID info record provides identity.
  - With `FAN_REPORT_PIDFD`: a pidfd info record may carry `FAN_EPIDFD` if the originating process has exited.
- `pid`: tgid by default; tid if `FAN_REPORT_TID`.
- Info records follow until `event_len` reached; consumers MUST iterate via `event_info_header.len`.

REQ-4: Per-permission response:
- Userspace writes one or more `struct fanotify_response` records to the fd.
- `fd` must match a pending event fd. Mismatched fd: write returns -EINVAL.
- `response` low 8 bits: FAN_ALLOW or FAN_DENY (plus FAN_AUDIT / FAN_INFO flags); upper 8 bits: errno when FAN_DENY (`FAN_DENY_ERRNO`).
- Default deny errno is EPERM if `FAN_DENY_ERRNO(0)`/plain FAN_DENY.
- Writing on a non-perm event fd: -EINVAL.

REQ-5: Per-FAN_ALLOW / FAN_DENY accounting:
- The originating syscall in the watched task remains blocked until response arrives or the group fd is closed.
- Per-`/proc/sys/fs/fanotify/max_queued_events` queue cap + per-group default fanotify_max_queued_events (unless FAN_UNLIMITED_QUEUE).
- Per-`/proc/sys/fs/fanotify/max_user_marks` per-UID mark cap (unless FAN_UNLIMITED_MARKS).
- Per-`/proc/sys/fs/fanotify/max_user_groups` per-UID instance cap.
- Per-perm-event timeout (`fanotify_perm_timeout`): on timeout, default response is FAN_ALLOW (configurable; per `fanotify_perm_response_timeout`).

REQ-6: Per-FAN_REPORT_FID/DIR_FID/NAME ordering:
- Each emitted event includes the smallest info-record set required by the active report flags.
- DFID_NAME records terminate the name with a single NUL and pad to 4-byte alignment within the info record's `len`.

REQ-7: Per-FAN_RENAME with FID flags:
- Emits one event with OLD_DFID_NAME and NEW_DFID_NAME info records (and optionally TARGET FID if `FAN_REPORT_TARGET_FID`).

REQ-8: Per-FAN_FS_ERROR:
- Emits exactly one event per error condition per superblock; subsequent identical errors merge into the same `event_info_error` with `error_count++`.
- The mark must be FILESYSTEM-scope.

REQ-9: Per-FAN_MARK_FLUSH:
- Removes all marks of the given scope (INODE / MOUNT / FILESYSTEM / MNTNS) for this group; mask argument is ignored.

REQ-10: Per-FAN_MARK_EVICTABLE:
- Only valid for INODE-scope marks on inodes; reclaimable via `shrink_active_list` under memory pressure; eviction emits no IN_IGNORED-style notification.

REQ-11: Per-FAN_REPORT_PIDFD:
- For each event, a `fanotify_event_info_pidfd` record carries a pidfd referring to the originating task.
- If task has already exited at event-emit time: pidfd field == `FAN_EPIDFD` (-2).
- If pidfd cannot be allocated: pidfd field == `FAN_NOPIDFD` (-1).

REQ-12: Per-FAN_REPORT_MNT:
- For `FAN_MNT_ATTACH` / `FAN_MNT_DETACH`, a `fanotify_event_info_mnt` record carries the mount id.

REQ-13: Per-FAN_PRE_ACCESS:
- Permission-class hook fired before the kernel performs a read from non-resident content (HSM); a `fanotify_event_info_range` provides the requested byte range. Response semantics same as other perm events.

REQ-14: Per-FAN_ENABLE_AUDIT:
- Permission responses recorded via the audit subsystem when `FAN_AUDIT` is set in the response and either `FAN_RESPONSE_INFO_NONE` or `FAN_RESPONSE_INFO_AUDIT_RULE` follows.

REQ-15: Per-event-fd lifetime:
- Each event delivered to userspace consumes one fd-table slot in the recipient. The recipient is responsible for `close(2)` on the event fd; failure to close throttles future events at `max_queued_events`.

REQ-16: Per-group close:
- All pending perm events are released with FAN_ALLOW (per default policy) at group destruction.
- All marks are destroyed; per-UID counters decremented.

REQ-17: Per-poll/epoll:
- `POLLIN` when event(s) pending or queue overflow flag set.
- `POLLOUT` when there is space to write at least one `fanotify_response`.

REQ-18: Per-FAN_EVENT_OK / FAN_EVENT_NEXT:
- ABI guarantees these macros remain valid for parsing buffers returned by `read(fd)`.

## Acceptance Criteria

- [ ] AC-1: `fanotify_init(0, 0)` without CAP_SYS_ADMIN: returns -EPERM (unless user-namespaced minimal mode allowed).
- [ ] AC-2: `fanotify_init(FAN_CLOEXEC | FAN_NONBLOCK | FAN_CLASS_NOTIF, O_RDONLY | O_CLOEXEC)`: returns fd ≥ 0.
- [ ] AC-3: `fanotify_init(FAN_CLASS_NOTIF | FAN_CLASS_CONTENT, ...)`: returns -EINVAL.
- [ ] AC-4: `fanotify_init(FAN_REPORT_NAME, ...)` without FAN_REPORT_DIR_FID: returns -EINVAL.
- [ ] AC-5: `fanotify_init(FAN_REPORT_TARGET_FID, ...)` without full DFID_NAME_TARGET set: returns -EINVAL.
- [ ] AC-6: `fanotify_init(FAN_UNLIMITED_QUEUE, ...)` without CAP_SYS_ADMIN: returns -EPERM.
- [ ] AC-7: `fanotify_init(FAN_ENABLE_AUDIT, ...)` without CAP_AUDIT_WRITE: returns -EPERM.
- [ ] AC-8: `fanotify_mark(fd, FAN_MARK_ADD | FAN_MARK_REMOVE, ...)`: returns -EINVAL.
- [ ] AC-9: `fanotify_mark(fd, FAN_MARK_ADD | FAN_MARK_ONLYDIR, ...)` on regular file: -ENOTDIR.
- [ ] AC-10: `fanotify_mark(fd, FAN_MARK_FLUSH | FAN_MARK_MOUNT, 0, AT_FDCWD, NULL)`: removes only MOUNT-scope marks.
- [ ] AC-11: Permission event response with mismatched fd: write returns -EINVAL.
- [ ] AC-12: Permission event FAN_DENY: originating syscall returns -EPERM (or encoded errno).
- [ ] AC-13: Permission event timeout: default FAN_ALLOW applied.
- [ ] AC-14: `read(fd)` of event metadata: `vers == 3`, `metadata_len == sizeof(struct fanotify_event_metadata)`, `reserved == 0`.
- [ ] AC-15: With FAN_REPORT_FID, event fd == FAN_NOFD and a TYPE_FID info record is present.
- [ ] AC-16: With FAN_REPORT_PIDFD, post-exit task yields pidfd field == FAN_EPIDFD.
- [ ] AC-17: Queue overflow emits exactly one event with `mask == FAN_Q_OVERFLOW`.
- [ ] AC-18: `FAN_FS_ERROR` repeats coalesce into `error_count` of the same `event_info_error`.
- [ ] AC-19: `FAN_RENAME` event carries both OLD_DFID_NAME and NEW_DFID_NAME info records.
- [ ] AC-20: `close(group_fd)` releases all marks and unblocks all pending perm events with FAN_ALLOW.
- [ ] AC-21: `FAN_EVENT_OK(meta, len)` is true iff buffer contains a complete record.

## Architecture

```
pub struct FanotifyGroup {
    pub flags:         FanotifyInitFlags,   // class + report toggles
    pub event_f_flags: u32,                 // O_* for event fds
    pub queue:         EventQueue,
    pub marks:         MarkSet,             // per-inode / mount / sb / mntns
    pub user:          Arc<UserStruct>,
    pub max_events:    u32,
    pub max_marks:     u32,
    pub perm_pending:  PermPendingMap,      // fd -> waiter
    pub perm_timeout:  Duration,
}

pub struct FanotifyMark {
    pub scope:        MarkScope,           // Inode | Mount | Filesystem | Mntns
    pub target:       MarkTarget,          // Arc<Inode> | Arc<Mount> | Arc<SuperBlock> | Arc<MntNamespace>
    pub mask:         u64,                 // event bits
    pub ignored_mask: u64,                 // FAN_MARK_IGNORE bits
    pub flags:        MarkFlags,           // EVICTABLE | IGNORED_SURV_MODIFY | ...
    pub group:        Weak<FanotifyGroup>,
}

pub struct FanotifyEventMetadata {
    pub event_len:    u32,
    pub vers:         u8,
    pub reserved:     u8,
    pub metadata_len: u16,
    pub mask:         u64,
    pub fd:           i32,
    pub pid:          i32,
}
```

`Fanotify::sys_init(flags, event_f_flags) -> Result<Fd>`:
1. /* Capability gates */
2. require_cap_or_user_ns(flags).
3. /* Mutually exclusive class */
4. classes = flags & (FAN_CLASS_NOTIF | FAN_CLASS_CONTENT | FAN_CLASS_PRE_CONTENT).
5. if classes.count_ones() > 1: return Err(EINVAL).
6. /* Report flag consistency */
7. if (flags & FAN_REPORT_NAME) && !(flags & FAN_REPORT_DIR_FID): return Err(EINVAL).
8. if (flags & FAN_REPORT_TARGET_FID) && (flags & FAN_REPORT_DFID_NAME_TARGET) != FAN_REPORT_DFID_NAME_TARGET: return Err(EINVAL).
9. /* Cap checks */
10. if (flags & FAN_UNLIMITED_QUEUE) && !capable(CAP_SYS_ADMIN): return Err(EPERM).
11. if (flags & FAN_UNLIMITED_MARKS) && !capable(CAP_SYS_ADMIN): return Err(EPERM).
12. if (flags & FAN_ENABLE_AUDIT) && !capable(CAP_AUDIT_WRITE): return Err(EPERM).
13. /* event_f_flags */
14. if event_f_flags & !ALLOWED_EVENT_F_FLAGS: return Err(EINVAL).
15. /* Per-UID counts */
16. if user.fanotify_groups >= sysctl_max_user_groups && !(flags & FAN_UNLIMITED_MARKS): return Err(EMFILE).
17. group = FanotifyGroup::new(flags, event_f_flags, user.clone()).
18. fd = anon_inode_fd("fanotify", &FANOTIFY_FOPS, group, flags & (O_CLOEXEC | O_NONBLOCK)).
19. user.fanotify_groups += 1.
20. return Ok(fd).

`Fanotify::sys_mark(fd, flags, mask, dirfd, pathname) -> Result<()>`:
1. action_bits = flags & (FAN_MARK_ADD | FAN_MARK_REMOVE | FAN_MARK_FLUSH).
2. if action_bits.count_ones() != 1: return Err(EINVAL).
3. scope = decode_scope(flags).ok_or(EINVAL)?.
4. if (flags & FAN_MARK_IGNORE) && (flags & FAN_MARK_IGNORED_MASK): return Err(EINVAL).
5. if scope != Inode && (flags & FAN_MARK_IGNORE) && !(flags & FAN_MARK_IGNORED_SURV_MODIFY): return Err(EINVAL).
6. if flags & FAN_MARK_EVICTABLE && scope != Inode: return Err(EINVAL).
7. group = fd_to_group(fd)?.
8. if !mask_compatible_with_class(mask, group.flags): return Err(EINVAL).
9. /* Resolve target */
10. target = resolve_mark_target(scope, dirfd, pathname, flags)?.
11. match action_bits {
        FAN_MARK_ADD => Fanotify::mark_add(group, target, mask, flags),
        FAN_MARK_REMOVE => Fanotify::mark_remove(group, target, mask, flags),
        FAN_MARK_FLUSH => Fanotify::mark_flush(group, scope),
    }

`Fanotify::file_read(file, buf, count) -> Result<usize>`:
1. group = file.private_data.
2. /* Wait if empty */
3. if group.queue.is_empty():
   - if file.flags & O_NONBLOCK: return Err(EAGAIN).
   - wait_event_interruptible(group.wait, !group.queue.is_empty()).
4. n = 0.
5. while let Some(ev) = group.queue.peek():
     - rec_len = ev.serialized_len(group.flags).
     - if n + rec_len > buf.len(): break.
     - group.queue.pop().
     - copy_to_user(buf[n..], ev.serialize(group.flags))?.
     - if ev.is_perm() && ev.fd != FAN_NOFD:
         group.perm_pending.insert(ev.fd, ev.waiter).
     - n += rec_len.
6. if n == 0: return Err(EINVAL).      /* buffer too small */
7. return Ok(n).

`Fanotify::file_write(file, buf, count) -> Result<usize>`:
1. group = file.private_data.
2. n = 0.
3. while n + sizeof::<FanotifyResponse>() <= count:
     - resp: FanotifyResponse = copy_from_user(buf[n..])?.
     - waiter = group.perm_pending.remove(resp.fd).ok_or(EINVAL)?.
     - if resp.response & FAN_AUDIT: audit_log(&waiter, resp).
     - if resp.response & FAN_INFO:
         info = copy_from_user::<FanotifyResponseInfoAuditRule>(buf[n + sizeof::<FanotifyResponse>()..])?.
         n += info.hdr.len as usize.
     - waiter.complete(resp.response).
     - n += sizeof::<FanotifyResponse>().
4. return Ok(n).

`Fanotify::handle_perm_event(target, mask) -> Result<()>`:
1. for mark in resolve_marks(target):
2.    if mark.mask & mask == 0: continue.
3.    waiter = PermWaiter::new(mask).
4.    group.queue.push_perm(waiter.clone(), open_event_fd(target, group.event_f_flags)?).
5.    /* Block */
6.    response = waiter.wait_timeout(group.perm_timeout).unwrap_or(FAN_ALLOW).
7.    if response & 0xFF == FAN_DENY:
        errno = (response >> FAN_ERRNO_SHIFT) & FAN_ERRNO_MASK;
        return Err(errno_or_eperm(errno)).
8. return Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `init_class_exclusive` | INVARIANT | sys_init: at most one of FAN_CLASS_NOTIF/CONTENT/PRE_CONTENT. |
| `init_report_consistency` | INVARIANT | FAN_REPORT_NAME requires FAN_REPORT_DIR_FID; FAN_REPORT_TARGET_FID requires full DFID_NAME_TARGET. |
| `init_cap_gates` | INVARIANT | UNLIMITED_QUEUE/MARKS require CAP_SYS_ADMIN; ENABLE_AUDIT requires CAP_AUDIT_WRITE. |
| `mark_action_exclusive` | INVARIANT | sys_mark: exactly one of ADD / REMOVE / FLUSH. |
| `mark_scope_exclusive` | INVARIANT | sys_mark: exactly one of INODE / MOUNT / FILESYSTEM / MNTNS. |
| `mark_ignore_exclusive` | INVARIANT | FAN_MARK_IGNORE incompatible with FAN_MARK_IGNORED_MASK. |
| `evictable_inode_only` | INVARIANT | FAN_MARK_EVICTABLE valid only for inode-scope marks. |
| `event_metadata_invariant` | INVARIANT | every emitted event has vers==3, reserved==0, metadata_len==sizeof. |
| `perm_response_fd_match` | INVARIANT | response.fd must match a pending event fd. |
| `perm_response_deny_errno_in_byte` | INVARIANT | FAN_DENY_ERRNO encodes errno only in the upper 8 bits. |
| `info_record_len_sum_matches_event_len` | INVARIANT | sum(info_record.len) + metadata_len == event_len. |
| `overflow_single_marker` | INVARIANT | per-group: at most one FAN_Q_OVERFLOW pending. |
| `pidfd_sentinels_only_on_failure` | INVARIANT | FAN_NOPIDFD on alloc fail; FAN_EPIDFD on exited task. |

### Layer 2: TLA+

`uapi/fanotify.tla`:
- Per-group lifecycle + per-mark add/remove/flush + per-event emit + per-perm-response wait + per-close.
- Properties:
  - `safety_class_singleton` — per-group: class never changes after init; exactly one class active.
  - `safety_perm_response_unblocks_waiter` — per-perm-event: FAN_ALLOW or FAN_DENY response unblocks the waiter exactly once.
  - `safety_perm_event_fd_lifecycle` — per-perm-event: event fd is opened once, closed exactly once (by recipient).
  - `safety_overflow_no_double_q_overflow` — per-group: FAN_Q_OVERFLOW marker emitted at most once between drains.
  - `safety_fs_error_coalesces` — per-superblock: identical FAN_FS_ERROR events increment error_count rather than duplicate.
  - `safety_mark_scope_singleton` — per-mark: scope assigned at creation and immutable.
  - `liveness_perm_event_eventually_resolved` — per-perm-event: response or timeout-default ALLOW within bounded time.
  - `liveness_close_drains_perm` — per-group-close: all pending perm waiters released with FAN_ALLOW.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Fanotify::sys_init` post: returns fd ∨ negative errno (cap + class + report consistency enforced) | `Fanotify::sys_init` |
| `Fanotify::sys_mark` post: action ∈ {ADD, REMOVE, FLUSH} ∧ scope ∈ {INODE, MOUNT, FILESYSTEM, MNTNS} | `Fanotify::sys_mark` |
| `Fanotify::file_read` post: every record satisfies FAN_EVENT_OK(buf+offset, remaining) | `Fanotify::file_read` |
| `Fanotify::file_write` post: each response consumes exactly one pending perm event | `Fanotify::file_write` |
| `Fanotify::handle_perm_event` post: blocks until response ∨ timeout; returns ALLOW or errno-encoded DENY | `Fanotify::handle_perm_event` |
| `Fanotify::mark_remove` post: mark removed iff mask becomes 0 ∧ ignored_mask becomes 0 | `Fanotify::mark_remove` |

### Layer 4: Verus/Creusot functional

`Per-fanotify_init(flags, event_f_flags) → per-fanotify_mark(fd, flags, mask, dirfd, path) → per-fsnotify-event → per-read(fd) → per-write(fd, response) → per-close(fd)` semantic equivalence: per-`Documentation/admin-guide/filesystem-monitoring.rst` + man-page `fanotify(7)` / `fanotify_init(2)` / `fanotify_mark(2)`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

fanotify(7) reinforcement:

- **Per-CAP_SYS_ADMIN required for FAN_CLASS_CONTENT / PRE_CONTENT** — defense against per-unprivileged blocking of privileged tasks.
- **Per-CAP_SYS_ADMIN for FAN_UNLIMITED_QUEUE / FAN_UNLIMITED_MARKS** — defense against per-resource-exhaustion DoS.
- **Per-CAP_AUDIT_WRITE for FAN_ENABLE_AUDIT** — defense against per-unauthorized audit-record-forge.
- **Per-FAN_REPORT_* flag-set self-consistency check** — defense against per-malformed-event-stream undefined-behavior.
- **Per-class-singleton enforcement at init** — defense against per-class confusion.
- **Per-event metadata vers==FANOTIFY_METADATA_VERSION** — defense against per-ABI-version skew.
- **Per-FAN_DENY errno encoded in upper 8 bits only** — defense against per-response-bit smuggling.
- **Per-perm-event default-allow on timeout** — defense against per-hang on dead userspace listener (configurable).
- **Per-perm-event close(group) ⇒ FAN_ALLOW all pending** — defense against per-group-leak indefinite block.
- **Per-info-record `len` self-consistent** — defense against per-malformed-record kernel infoleak.
- **Per-`fanotify_event_info_fid.handle` capped to f_handle MAX** — defense against per-handle-length overflow.
- **Per-FAN_REPORT_FID drops event-fd; userspace uses open_by_handle_at** — defense against per-fd-table exhaustion.
- **Per-FAN_MARK_EVICTABLE limited to inode marks** — defense against per-stale-mount/sb mark trash.
- **Per-FAN_NOPIDFD / FAN_EPIDFD sentinels for alloc-fail/exited-task** — defense against per-userspace dereference of stale pidfd.
- **Per-NAME zero-pad in DFID_NAME** — defense against per-stack/heap padding infoleak.
- **Per-`/proc/sys/fs/fanotify/*` sysctl caps** — defense against per-DoS via uncapped marks/queues.
- **Per-rcu-free of group / marks** — defense against per-UAF on concurrent close.
- **Per-perm-response audit-log on FAN_AUDIT** — defense against per-silent policy decisions.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF / USERCOPY**: every `pathname` arg to `sys_fanotify_mark`, every `read(fd, __user *buf, ...)`, every `write(fd, __user *resp, ...)` and every variable info record copied to userspace is mediated by `copy_from_user` / `copy_to_user` with explicit length checks. No raw `__user` deref in event-emit paths (`fanotify_event_info_fid.handle[]` is built kernel-side and `copy_to_user`'d in one shot).
- **PAX_RANDKSTACK**: per-syscall entry of `fanotify_init` / `fanotify_mark` randomizes the kernel-stack offset so that pointer-spray-into-`struct fanotify_group` reconnaissance against the perm-event-blocking task is defeated.
- **GRKERNSEC_PROC restriction on /proc/<pid>/fdinfo/<N>**: fanotify `fdinfo` exposes lines like `fanotify ino:<inode> sdev:<dev> mflags:<mark flags> mask:<...> ignored_mask:<...> fhandle-bytes:<...> fhandle-type:<...> f_handle:<hex>`. Under `GRKERNSEC_PROC_USER` / `_USERGROUP`, this leaks watched-inode FIDs equivalent to `open_by_handle_at(2)` arguments; restrict to the owning UID (or CAP_SYS_PTRACE). Apply the same gate to `/proc/<pid>/fdinfo` of any task that holds a fanotify event fd.
- **GRKERNSEC_HARDEN_PTRACE — fanotify perm-mode requires CAP_SYS_ADMIN**: even after the LSM `security_fanotify_init` hook, harden by forbidding non-`CAP_SYS_ADMIN` callers from creating `FAN_CLASS_CONTENT` / `FAN_CLASS_PRE_CONTENT` / `FAN_REPORT_PIDFD` groups inside a user namespace (independent of UID-mapping); these flags expose the ability to block syscalls of privileged tasks and to obtain pidfds for processes outside the caller's confined view. `ptrace_may_access` gates per-event fd handoff.
- **GRKERNSEC_NO_FIFO interaction with FAN_FS_ERROR**: when `GRKERNSEC_NO_FIFO` denies opening FIFOs in non-root-owned `/tmp`-like dirs, the resulting EPERM/EACCES surface MUST NOT be reported via `FAN_FS_ERROR` as a "filesystem error" — only true storage-corruption / journal / metadata errors are reported. `errno` for grsec-policy denies is filtered out of `fanotify_event_info_error.error` to prevent unprivileged listeners from inferring grsec policy state.
- **FAN_UNLIMITED_QUEUE / FAN_UNLIMITED_MARKS treated as security-hostile**: in addition to CAP_SYS_ADMIN, harden by `GRKERNSEC_DENY_USB`-style policy file under `/sys/kernel/grsecurity/fanotify_unlimited` that requires explicit operator opt-in to allow any caller, even root, to use these flags (default: deny).
- **inotify IN_ATTRIB on protected files (chattr +i / +a) — covered by PaX UDEREF**: equivalent FAN_ATTRIB events on `S_IMMUTABLE` / `S_APPEND` inodes are gated to require `CAP_LINUX_IMMUTABLE` on the listener; UDEREF additionally validates the `event_info_fid.handle[]` user-copy boundary so a malicious listener cannot induce an OOB read of the inode's `file_handle`.
- **Landlock as a complement to grsec PaX MPROTECT**: a fanotify-restricted unprivileged process should also be landlock-restricted to the directory subtree it intends to monitor — the kernel refuses `fanotify_mark` calls whose `dirfd`/`pathname` resolution escapes the calling Landlock domain (per `security_path_*` LSM hooks). PaX MPROTECT and Landlock together force unprivileged confined monitors to be observers, not enforcers.
- **Landlock as namespace-confining — additive to user-namespace gates**: an unprivileged user namespace with a Landlock domain cannot create `FAN_CLASS_CONTENT` groups even if its `CAP_SYS_ADMIN` bit appears set within the namespace; the user-namespace cap is treated as insufficient on its own.
- **Per-fd close-on-exec mandatory**: hardened builds reject `fanotify_init(flags, ...)` where `flags & FAN_CLOEXEC == 0` (mirror of the grsec-`GRKERNSEC_HARDEN_EXEC` policy for inotify); likewise the per-event fd's `event_f_flags` must contain `O_CLOEXEC`.
- **Per-real-UID accounting** for `fanotify_groups` / `fanotify_marks` (not effective UID); defense against setuid-escalation circumvention of caps.
- **`f_handle` randomized salt**: `fanotify_event_info_fid.handle[]` is opaque to userspace but kernel-side derived from a per-superblock fhandle key; grsec rotates that key on superblock mount so leaked FIDs do not survive a remount. Defends against per-long-lived-FID replay attacks via `open_by_handle_at`.
- **Audit-log every FAN_DENY on FAN_CLASS_PRE_CONTENT**: defends against per-silent HSM-policy denial; grsec auditing is independent of `FAN_ENABLE_AUDIT` opt-in.
- **`fanotify_response` write rate-limited per group** — defense against per-response-flood reordering attacks.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- inotify (covered in `uapi/headers/inotify.md` Tier-5)
- Landlock LSM (covered in `uapi/headers/landlock.md` Tier-5)
- fsnotify common backbone (covered in `fs/notify.md` Tier-3)
- audit-subsystem internals (covered in `kernel/audit.md` Tier-3)
- open_by_handle_at / name_to_handle_at (covered separately if expanded)
- pidfd_open / pidfd_send_signal (covered in `kernel/pidfd.md` Tier-3)
- man-page `fanotify(7)` / `fanotify_init(2)` / `fanotify_mark(2)` prose (upstream copy)
- Implementation code
