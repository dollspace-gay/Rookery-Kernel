# Tier-5: syscall 245 — mq_getsetattr(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`245  common  mq_getsetattr  sys_mq_getsetattr`)
  - ipc/mqueue.c (`SYSCALL_DEFINE3(mq_getsetattr, ...)`, `do_mq_getsetattr`)
  - include/uapi/linux/mqueue.h (`struct mq_attr`)
  - include/uapi/asm-generic/fcntl.h (`O_NONBLOCK`)
-->

## Summary

`mq_getsetattr(2)` is **x86_64 syscall 245**, the combined POSIX message-queue attribute getter/setter. It is the single kernel entry point underlying the glibc helpers `mq_getattr(3)` and `mq_setattr(3)`. Given an open mqueue descriptor `mqdes`, the call may *atomically*:

1. Replace the descriptor's mutable flags from `*newattr` (only `O_NONBLOCK` is mutable; all other fields in `newattr` are ignored), and
2. Return the prior `struct mq_attr` snapshot (including the immutable `mq_maxmsg`/`mq_msgsize` and live `mq_curmsgs`) into `*oldattr`.

Either pointer may be `NULL`: if `newattr == NULL` the call is a pure get; if `oldattr == NULL` the call discards the snapshot and just updates flags. Both `NULL` is valid (a no-op, returns `0`).

Critical for: every glibc `mq_getattr`/`mq_setattr` call, every queue introspection (current depth, max depth, msg size), every dynamic switch between blocking and non-blocking I/O on an existing mqueue descriptor, every Rookery test for the strict `O_NONBLOCK`-only mutability rule.

## Signature

C (POSIX / man-pages):

```c
int mq_getsetattr(mqd_t mqdes, const struct mq_attr *newattr,
                  struct mq_attr *oldattr);
```

glibc wrappers:

```c
int mq_getattr(mqd_t mqdes, struct mq_attr *attr);            /* -> mq_getsetattr(mqdes, NULL, attr) */
int mq_setattr(mqd_t mqdes, const struct mq_attr *new,
               struct mq_attr *old);                            /* -> mq_getsetattr(mqdes, new, old) */
```

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(mq_getsetattr,
                mqd_t,                       mqdes,
                const struct mq_attr __user *, u_mqstat,
                struct mq_attr __user *,       u_omqstat);
```

Rookery dispatch:

```rust
pub fn sys_mq_getsetattr(
    mqdes: i32,
    newattr: UserPtr<MqAttr>,
    oldattr: UserPtr<MqAttr>,
) -> SyscallResult<i32>;
```

## Parameters

| name     | type                          | constraints                                                                                       | errno-on-bad           |
|----------|-------------------------------|---------------------------------------------------------------------------------------------------|------------------------|
| mqdes    | `mqd_t` (`int`)               | Open mqueue fd; access mode of the fd is not constrained for this call.                            | `EBADF`/`EBADMSG`      |
| newattr  | `const struct mq_attr __user *` | May be `NULL`. If non-NULL, readable for `sizeof(struct mq_attr)`. Only `mq_flags & O_NONBLOCK` honored; remaining fields and reserved zero-padding ignored. | `EFAULT`/`EINVAL`     |
| oldattr  | `struct mq_attr __user *`     | May be `NULL`. If non-NULL, writable for `sizeof(struct mq_attr)`.                                | `EFAULT`               |

`struct mq_attr` layout (UAPI):

```c
struct mq_attr {
    long mq_flags;     /* mutable: O_NONBLOCK only */
    long mq_maxmsg;    /* immutable: from create-time */
    long mq_msgsize;   /* immutable: from create-time */
    long mq_curmsgs;   /* live: number of messages currently queued */
    long __reserved[4];
};
```

## Return value

- Success: `0`.
- Failure: `-1` with `errno`; in-kernel: a negated errno.

## Errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EBADF`        | `mqdes` is not a valid descriptor.                                                              |
| `EBADMSG`      | `mqdes` refers to a non-mqueue file.                                                            |
| `EFAULT`       | `newattr` or `oldattr` (when non-NULL) crosses the user/kernel boundary illegally.              |
| `EINVAL`       | `newattr` non-NULL and `newattr->mq_flags & ~O_NONBLOCK != 0`. (Strict mask rejection.)         |

## ABI surface (constants + flags)

From `include/uapi/asm-generic/fcntl.h` and `include/uapi/linux/mqueue.h`:

- `O_NONBLOCK = 0o4000` — only bit in `mq_flags` that `mq_setattr` may toggle.
- `struct mq_attr` size: `8 * sizeof(long)` (4 user fields + 4 reserved).

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=mqdes`, `%rsi=newattr`, `%rdx=oldattr`.
- REQ-2: `fget(mqdes)`; `f->f_op != &mqueue_file_operations ⟹ -EBADMSG`. No `FMODE_*` check (descriptor need not be writable or readable).
- REQ-3: If `newattr != NULL`: `copy_from_user(&kattr, newattr, sizeof(mq_attr))?`.
- REQ-4: `kattr.mq_flags & ~O_NONBLOCK != 0 ⟹ -EINVAL`. All other `kattr` fields ignored (no validation, no use).
- REQ-5: If `oldattr != NULL`: read pre-update snapshot inside `info->lock`:
  - `omq.mq_flags = (f->f_flags & O_NONBLOCK)`,
  - `omq.mq_maxmsg = info->attr.mq_maxmsg`,
  - `omq.mq_msgsize = info->attr.mq_msgsize`,
  - `omq.mq_curmsgs = info->attr.mq_curmsgs`,
  - reserved fields zeroed.
- REQ-6: If `newattr != NULL`: atomically (still under `info->lock` or `f->f_lock`) update `f->f_flags`:
  - `f->f_flags = (f->f_flags & ~O_NONBLOCK) | (kattr.mq_flags & O_NONBLOCK)`.
- REQ-7: Release lock; if `oldattr != NULL` ⟹ `copy_to_user(oldattr, &omq, sizeof(mq_attr))?`.
- REQ-8: Snapshot must be taken *before* the update (oldattr reflects the prior `O_NONBLOCK`).
- REQ-9: Atomicity: get-then-set ordering observable; reads from a concurrent thread must not see a torn `struct mq_attr` (snapshot is local then copied out).
- REQ-10: No effect on queue contents, registration, or accounting — flags-only.
- REQ-11: Audit hook: optional record on POSIX_MQ_SETATTR with old and new flags.

## Acceptance Criteria

- [ ] AC-1: `mq_getattr(fd, &a)` returns 0; `a.mq_maxmsg`/`mq_msgsize` match create-time; `a.mq_curmsgs` reflects live count.
- [ ] AC-2: `mq_setattr(fd, &{mq_flags=O_NONBLOCK}, NULL)` flips `f_flags` to non-blocking; subsequent full-queue `mq_timedsend` ⟹ `-EAGAIN`.
- [ ] AC-3: `mq_setattr(fd, &{mq_flags=0}, NULL)` clears `O_NONBLOCK`; send blocks as usual.
- [ ] AC-4: `mq_setattr(fd, &{mq_flags=O_NONBLOCK|O_APPEND}, NULL)` ⟹ `-EINVAL`.
- [ ] AC-5: `mq_setattr(fd, &new, &old)` returns prior `O_NONBLOCK` in `old.mq_flags`.
- [ ] AC-6: `mq_getsetattr(fd, NULL, NULL)` ⟹ `0`, no effect.
- [ ] AC-7: Non-mqueue fd ⟹ `-EBADMSG`.
- [ ] AC-8: Invalid fd ⟹ `-EBADF`.
- [ ] AC-9: `oldattr` write fault ⟹ `-EFAULT`; *but* update of `f_flags` already applied (POSIX-permitted; matches Linux). [DESIGN NOTE: this mirrors upstream behavior — document explicitly.]
- [ ] AC-10: Concurrent send/receive does not corrupt the `mq_curmsgs` snapshot returned in `oldattr`.
- [ ] AC-11: Attempt to change `mq_maxmsg` or `mq_msgsize` via `newattr` is silently ignored (no error, no change), per POSIX.
- [ ] AC-12: Cross-thread `mq_getsetattr` ordering: two threads' updates serialized by `f_lock`/`info->lock`; final state is one of the two.

## Architecture

```
struct MqGetsetattrArgs {
    mqdes: i32,
    newattr: UserPtr<MqAttr>,
    oldattr: UserPtr<MqAttr>,
}
```

`Mqueue::sys_mq_getsetattr(args) -> i32`:

1. `let file = current().files.fget(args.mqdes)?;`
2. `if !Mqueue::is_mqueue_file(&file) { return -EBADMSG; }`
3. `let info = mqueue_inode_info(file.inode);`
4. `let newk = if !args.newattr.is_null() { let a = MqAttr::copy_from_user(args.newattr)?; if a.mq_flags & !(O_NONBLOCK as i64) != 0 { return -EINVAL; } Some(a) } else { None };`
5. `let snap = Mqueue::lock_and_update(&file, info, newk);`
6. `if !args.oldattr.is_null() { snap.copy_to_user(args.oldattr)?; }`
7. `return 0;`

`Mqueue::lock_and_update(file, info, newk) -> MqAttr`:

1. `info.lock.lock();`
2. `let snap = MqAttr {`
3. `    mq_flags:   (file.f_flags as i64) & (O_NONBLOCK as i64),`
4. `    mq_maxmsg:  info.attr.mq_maxmsg,`
5. `    mq_msgsize: info.attr.mq_msgsize,`
6. `    mq_curmsgs: info.attr.mq_curmsgs,`
7. `    __reserved: [0; 4],`
8. `};`
9. `if let Some(n) = newk {`
10. `    let new_flags = (file.f_flags & !O_NONBLOCK) | ((n.mq_flags as u32) & O_NONBLOCK);`
11. `    file.f_flags.store(new_flags, Ordering::Release);`
12. `}`
13. `info.lock.unlock();`
14. `return snap;`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mq_flags_mask_strict` | INVARIANT | `newattr->mq_flags & ~O_NONBLOCK != 0 ⟹ -EINVAL`. |
| `immutable_fields_ignored` | INVARIANT | `mq_maxmsg`/`mq_msgsize`/`mq_curmsgs` in `newattr` ignored. |
| `snapshot_before_update` | INVARIANT | `oldattr` reflects prior flags, not post-update. |
| `f_op_identity` | INVARIANT | Non-mqueue fd ⟹ `-EBADMSG`. |
| `lock_held_for_read_modify_write` | INVARIANT | `info->lock` (or `f_lock`) covers both snapshot and update. |
| `reserved_zeroed` | INVARIANT | `oldattr->__reserved[..]` returned as zero. |

### Layer 2: TLA+

`uapi/syscalls/mq_getsetattr.tla`:
- Per-call → fget → optional copy_from_user(newattr) → lock-snapshot-update → unlock → optional copy_to_user(oldattr).
- Properties:
  - `safety_only_O_NONBLOCK_mutated`,
  - `safety_snapshot_pre_update`,
  - `safety_atomic_visibility`,
  - `liveness_mq_getsetattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ `f_flags` matches `(orig & ~O_NONBLOCK) | (new & O_NONBLOCK)` when `newattr` non-NULL | `Mqueue::lock_and_update` |
| Post: success ⟹ `oldattr` (if non-NULL) reflects pre-update flags + live attrs | `Mqueue::lock_and_update` |
| Post: queue contents and registration unchanged | `Mqueue::sys_mq_getsetattr` |
| Post: any error ⟹ `f_flags` unchanged | `Mqueue::sys_mq_getsetattr` |

### Layer 4: Verus/Creusot functional

`mq_getsetattr(mqdes, new, old)` ≡ POSIX.1-2017 combined `mq_getattr`/`mq_setattr` per `man 3 mq_getattr`/`man 3 mq_setattr`; only `O_NONBLOCK` writable.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mq_getsetattr(2)` reinforcement:

- **Per-`mq_flags` strict mask** — defense against silent acceptance of unknown flag bits.
- **Per-immutable-field discipline** — defense against post-create geometry mutation (out-of-bounds enqueue).
- **Per-snapshot-before-update ordering** — defense against torn `oldattr` reads.
- **Per-`f_op` identity check** — defense against fd-confusion.
- **Per-`info->lock` scope** — defense against TOCTOU on concurrent send/recv.
- **Per-reserved-field zeroing** — defense against kernel-stack info leak.
- **No-access-mode requirement** — preserved; descriptor opened `O_RDONLY` can still query attrs (POSIX-required).

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `newattr`/`oldattr` SMAP-guarded; copy bounded by `sizeof(struct mq_attr)`.
- **GRKERNSEC_CHROOT_MQUEUE** — chrooted task can only query/modify attrs of queues in its IPC namespace and chroot-visible mqueue mount.
- **GRKERNSEC_AUDIT_IPC** — every `mq_setattr` (flag change) logged with mqdes/old-flags/new-flags/uid/exe.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_REFCOUNT** — `file`/`mqueue_inode_info` refcounts saturating.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — local `MqAttr` snapshot zeroed on stack on exit (defense against later stack-reuse leak).
- **GRKERNSEC_PROC_IPC** — `/proc/<pid>/fdinfo/<fd>` for mqueue does not leak full `mq_attr` (numbers only, no kernel addresses).
- **GRKERNSEC_KSTACKOVERFLOW** — fixed-size `mq_attr` stack copy verified within kernel stack canary range.
- **GRKERNSEC_RESLOG** — repeated `O_NONBLOCK` flips at high frequency logged as suspected polling-DoS.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `mq_open(2)` / `mq_unlink(2)` — separate Tier-5.
- `mq_timedsend(2)` / `mq_timedreceive(2)` / `mq_notify(2)` — separate Tier-5.
- mqueue filesystem internals — covered in `ipc/mqueue.md` Tier-3 if expanded.
- File-flags mutability in general (`fcntl(F_SETFL)`) — covered in `fcntl.md` Tier-5.
- Implementation code.
