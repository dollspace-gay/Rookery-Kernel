# Tier-5: syscall 240 — mq_open(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`240  common  mq_open  sys_mq_open`)
  - ipc/mqueue.c (`SYSCALL_DEFINE4(mq_open, ...)`, `do_mq_open`, `mqueue_create`, `mqueue_create_attr`, `mqueue_get_inode`)
  - include/uapi/linux/mqueue.h (`struct mq_attr`, `MQ_PRIO_MAX`, `MQ_BYTES_MAX`)
  - include/linux/mqueue.h (kernel `struct mqueue_inode_info`)
  - fs/namei.c (`filename_lookup`, `LOOKUP_DIRECTORY`)
  - kernel/sysctl.c (`fs.mqueue.queues_max`, `fs.mqueue.msg_max`, `fs.mqueue.msgsize_max`)
-->

## Summary

`mq_open(2)` is **x86_64 syscall 240**, the POSIX message-queue creator/opener. It returns a `mqd_t` (a file descriptor) referencing the message queue named `name` (which must begin with `/` and contain no further `/`). The call is the IPC analogue of `open(2)`: `oflag` selects open semantics (`O_RDONLY` / `O_WRONLY` / `O_RDWR`, optionally with `O_CREAT`, `O_EXCL`, `O_NONBLOCK`, `O_CLOEXEC`), and on first creation `mode` and `attr` set permissions and queue geometry.

Message queues live in a kernel-internal filesystem (`mqueue`) typically mounted at `/dev/mqueue`. Each queue is an inode whose payload is a doubly-bounded ring of priority-ordered messages: `attr.mq_maxmsg` messages of up to `attr.mq_msgsize` bytes each, with `MQ_PRIO_MAX = 32768` priority levels and a hard cap of `RLIMIT_MSGQUEUE` bytes of accounted storage per real-uid. Per-namespace `fs.mqueue.queues_max` (default 256) bounds the total number of queues per IPC namespace.

Critical for: every POSIX-RT messaging path (POSIX.1b `mq_*`), every async-notification consumer (`mq_notify`), every container-runtime that exposes `/dev/mqueue`, every Rookery IPC test for mqueue accounting and namespacing.

## Signature

C (POSIX / man-pages):

```c
mqd_t mq_open(const char *name, int oflag);
mqd_t mq_open(const char *name, int oflag, mode_t mode, struct mq_attr *attr);
```

glibc wrapper: `__mq_open` → `INLINE_SYSCALL(mq_open, 4, name, oflag, mode, attr)` (the 2-arg form passes `0` for mode/attr).

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE4(mq_open,
                const char __user *,    u_name,
                int,                    oflag,
                umode_t,                mode,
                struct mq_attr __user *,u_attr);
```

Rookery dispatch:

```rust
pub fn sys_mq_open(
    name: UserPtr<u8>,
    oflag: i32,
    mode: u16,
    attr: UserPtr<MqAttr>,
) -> SyscallResult<i32>;
```

## Parameters

| name  | type                      | constraints                                                                                       | errno-on-bad           |
|-------|---------------------------|---------------------------------------------------------------------------------------------------|------------------------|
| name  | `const char __user *`     | NUL-terminated; `strlen(name) ≤ MQ_NAME_MAX` (~`NAME_MAX - 1`); must start with `/` and contain no further `/`. | `EINVAL`/`ENAMETOOLONG`/`EFAULT` |
| oflag | `int`                     | Access mode `O_RDONLY`/`O_WRONLY`/`O_RDWR` plus optional `O_CREAT`/`O_EXCL`/`O_NONBLOCK`/`O_CLOEXEC`. | `EINVAL`              |
| mode  | `umode_t`                 | Only used when `O_CREAT` set; ANDed with `~current_umask()`.                                       | (n/a)                  |
| attr  | `struct mq_attr __user *` | Used only with `O_CREAT`; `attr->mq_maxmsg` and `attr->mq_msgsize` positive; bounded by `fs.mqueue.*_max`. | `EINVAL`/`EFAULT`     |

`struct mq_attr` layout (UAPI):

```c
struct mq_attr {
    long mq_flags;     /* set by mq_setattr (O_NONBLOCK) */
    long mq_maxmsg;    /* requested max messages */
    long mq_msgsize;   /* requested max message size */
    long mq_curmsgs;   /* current messages (set by mq_getattr) */
    long __reserved[4];
};
```

## Return value

- Success: a non-negative `mqd_t` (an integer file descriptor).
- Failure: `-1` with `errno`; in-kernel: a negated errno.

## Errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EFAULT`       | `name` or `attr` crosses the user/kernel boundary illegally.                                    |
| `EINVAL`       | `name` does not start with `/`, contains additional `/`, is empty, or only `/`; or `oflag` has unknown bits; or `attr->mq_maxmsg`/`mq_msgsize` non-positive or exceed sysctl maxima. |
| `ENAMETOOLONG` | `strlen(name) > NAME_MAX` for the mqueue filesystem.                                            |
| `EACCES`       | Caller lacks permission for the existing queue; or `O_CREAT` requested on a queue the caller cannot create. |
| `EEXIST`       | `O_CREAT | O_EXCL` and queue already exists.                                                    |
| `ENOENT`       | `O_CREAT` not set and queue does not exist.                                                     |
| `ENOMEM`       | Kernel could not allocate inode/queue storage.                                                  |
| `ENOSPC`       | `fs.mqueue.queues_max` would be exceeded for this IPC namespace; or `RLIMIT_MSGQUEUE` would be exceeded. |
| `EMFILE`       | Per-process file-descriptor limit reached.                                                       |
| `ENFILE`       | System-wide file-table exhausted.                                                                |

## ABI surface (constants + flags)

From `include/uapi/linux/mqueue.h` and `include/uapi/asm-generic/fcntl.h`:

- `MQ_PRIO_MAX = 32768` — number of priority levels (0..32767).
- `MQ_BYTES_MAX = 819200` — kernel default cap on queue accounted bytes (subject to `RLIMIT_MSGQUEUE`).
- `O_RDONLY = 0`, `O_WRONLY = 1`, `O_RDWR = 2`.
- `O_CREAT = 0o100`, `O_EXCL = 0o200`, `O_NONBLOCK = 0o4000`, `O_CLOEXEC = 0o2000000`.
- `fs.mqueue.queues_max` — default 256, max 65536 per IPC ns.
- `fs.mqueue.msg_max` — default 10, max bounded by HARD_MSGMAX (65536).
- `fs.mqueue.msgsize_max` — default 8192, max bounded by HARD_MSGSIZEMAX.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=name`, `%rsi=oflag`, `%rdx=mode`, `%r10=attr`.
- REQ-2: `name` must begin with `/`; absence ⟹ `-EINVAL`. A second `/` anywhere in the name ⟹ `-EINVAL`. Empty (`""`) or `"/"` alone ⟹ `-EINVAL`.
- REQ-3: `oflag & ~(O_ACCMODE | O_CREAT | O_EXCL | O_NONBLOCK | O_CLOEXEC) ⟹ -EINVAL`.
- REQ-4: `O_CREAT` set ⟹ `attr` may be `NULL` (use sysctl defaults) or pointer to valid `mq_attr` with both `mq_maxmsg ∈ (0, fs.mqueue.msg_max]` and `mq_msgsize ∈ (0, fs.mqueue.msgsize_max]`.
- REQ-5: `O_CREAT | O_EXCL` and queue exists ⟹ `-EEXIST`.
- REQ-6: New-queue accounting: `bytes = mq_maxmsg * (mq_msgsize + sizeof(struct msg_msg))` charged against caller `RLIMIT_MSGQUEUE` (and against namespace `queues_max`).
- REQ-7: Permissions: existing queue inode mode checked against caller's fsuid/fsgid; `O_RDONLY` needs read perm, `O_WRONLY` needs write perm, `O_RDWR` needs both.
- REQ-8: On success: returns lowest-available fd; `FD_CLOEXEC` set iff `O_CLOEXEC`; queue refcount incremented.
- REQ-9: `O_NONBLOCK` recorded in `f->f_flags` and on first creation in `mq_flags`.
- REQ-10: Per-IPC-namespace isolation: `mq_open` only sees queues created in the caller's IPC ns; `clone(CLONE_NEWIPC)` gives a fresh mqueue root.
- REQ-11: Queue inode created via `mqueue_create_attr`; backing `struct mqueue_inode_info` allocated under the mqueue superblock.
- REQ-12: Audit hook emitted for creation (POSIX_MQ class).

## Acceptance Criteria

- [ ] AC-1: `mq_open("/q", O_CREAT|O_RDWR, 0600, NULL)` returns valid fd; `/dev/mqueue/q` visible.
- [ ] AC-2: `mq_open("q", ...)` (no leading `/`) ⟹ `-EINVAL`.
- [ ] AC-3: `mq_open("/a/b", ...)` (interior `/`) ⟹ `-EINVAL`.
- [ ] AC-4: `mq_open("/q", O_CREAT|O_EXCL, ...)` on existing queue ⟹ `-EEXIST`.
- [ ] AC-5: `mq_open("/q", O_RDONLY)` without `O_CREAT` on missing queue ⟹ `-ENOENT`.
- [ ] AC-6: Exceeding `fs.mqueue.queues_max` ⟹ `-ENOSPC`.
- [ ] AC-7: Exceeding `RLIMIT_MSGQUEUE` for caller ⟹ `-ENOSPC`.
- [ ] AC-8: `attr->mq_maxmsg = 0` ⟹ `-EINVAL`.
- [ ] AC-9: `attr->mq_msgsize > fs.mqueue.msgsize_max` ⟹ `-EINVAL`.
- [ ] AC-10: `O_CLOEXEC` honored: `fcntl(fd, F_GETFD) & FD_CLOEXEC == FD_CLOEXEC`.
- [ ] AC-11: `O_NONBLOCK` recorded; `mq_getattr` reflects `mq_flags & O_NONBLOCK`.
- [ ] AC-12: Different IPC namespaces ⟹ same name yields independent queues.

## Architecture

```
struct MqOpenArgs {
    name:  UserPtr<u8>,
    oflag: i32,
    mode:  u16,
    attr:  UserPtr<MqAttr>,
}
```

`Mqueue::sys_mq_open(args) -> i32`:

1. /* Copy name */ `let kname = NameBuf::copy_from_user(args.name, NAME_MAX + 1)?;`
2. `if !Mqueue::valid_name(&kname) { return -EINVAL; }`
3. /* Validate oflag */ `if args.oflag & !(O_ACCMODE | O_CREAT | O_EXCL | O_NONBLOCK | O_CLOEXEC) != 0 { return -EINVAL; }`
4. /* Optionally copy attr */ `let kattr = if (args.oflag & O_CREAT) != 0 && !args.attr.is_null() { Some(MqAttr::copy_from_user(args.attr)?) } else { None };`
5. `if let Some(a) = &kattr { Mqueue::validate_attr(a)?; }`
6. `return Mqueue::do_open(&kname, args.oflag, args.mode & !current_umask(), kattr);`

`Mqueue::do_open(name, oflag, mode, attr) -> i32`:

1. `let ns = current_ipc_ns();`
2. `let root = ns.mq_mnt.root;`
3. `let dentry = filename_lookup(root, name, LOOKUP_DIRECTORY)?;`
4. `if (oflag & O_CREAT) != 0 { /* create-path */ Mqueue::create(ns, &dentry, mode, attr.as_ref(), oflag) } else { Mqueue::lookup_and_open(&dentry, oflag) }`

`Mqueue::create(ns, dentry, mode, attr, oflag) -> i32`:

1. `if dentry.exists() { return if (oflag & O_EXCL) != 0 { -EEXIST } else { Mqueue::lookup_and_open(dentry, oflag) }; }`
2. `let req_msg = attr.map(|a| a.mq_maxmsg).unwrap_or(ns.mq_msg_default);`
3. `let req_size = attr.map(|a| a.mq_msgsize).unwrap_or(ns.mq_msgsize_default);`
4. `Mqueue::charge_user(current_user(), req_msg, req_size)?;`
5. `let inode = Mqueue::get_inode(ns, mode, req_msg, req_size)?;`
6. `Mqueue::link_inode(dentry, inode)?;`
7. `return Mqueue::install_fd(inode, oflag);`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `name_leading_slash` | INVARIANT | `name[0] == '/'` required; else `-EINVAL`. |
| `name_no_interior_slash` | INVARIANT | No `/` in `name[1..]`. |
| `oflag_mask_strict` | INVARIANT | `oflag` only sets recognized bits. |
| `attr_bounds_enforced` | INVARIANT | `mq_maxmsg ∈ (0, msg_max]`, `mq_msgsize ∈ (0, msgsize_max]`. |
| `rlimit_msgqueue_charged` | INVARIANT | Bytes charged against caller's `RLIMIT_MSGQUEUE`. |
| `ipc_ns_isolated` | INVARIANT | Lookup confined to caller's IPC ns's mqueue root. |

### Layer 2: TLA+

`uapi/syscalls/mq_open.tla`:
- Per-call → name-validate → ns-resolve → create-or-lookup → fd-install.
- Properties:
  - `safety_excl_excludes_existing`,
  - `safety_namespace_isolation`,
  - `safety_rlimit_bound_observed`,
  - `liveness_mq_open_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ fd points to mqueue inode in current IPC ns | `Mqueue::do_open` |
| Post: `O_CREAT|O_EXCL` success ⟹ inode newly allocated | `Mqueue::create` |
| Post: any error ⟹ no fd consumed, no inode persists | `Mqueue::do_open` |
| Post: RLIMIT exceed ⟹ `-ENOSPC` and no inode allocated | `Mqueue::charge_user` |

### Layer 4: Verus/Creusot functional

`mq_open(name, oflag, mode, attr)` ≡ POSIX.1-2017 mq_open semantics per `man 3 mq_open` and `Documentation/admin-guide/sysctl/fs.rst` (mqueue section).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mq_open(2)` reinforcement:

- **Per-`name` syntactic gate** — defense against directory-traversal / arbitrary-path mqueue inodes.
- **Per-`oflag` mask check** — defense against silent forward-compat of unknown bits.
- **Per-`attr` bounds vs sysctl** — defense against runaway queue allocation.
- **Per-`RLIMIT_MSGQUEUE` charge** — defense against per-user memory exhaustion.
- **Per-`queues_max` enforcement** — defense against namespace-wide DoS.
- **Per-IPC-namespace scoping** — defense against cross-container queue discovery.
- **Per-`O_CLOEXEC` plumbed to fd table** — defense against fd-leak through exec.
- **Per-umask applied to mode** — defense against world-writable queues by default.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `name` and `attr` SMAP-guarded copies; `strncpy_from_user` bounded by `NAME_MAX`.
- **GRKERNSEC_CHROOT_MQUEUE** — `mq_open` inside chroot only sees queues created within the same chroot subtree (mqueue mount confined).
- **GRKERNSEC_RLIMIT** — `RLIMIT_MSGQUEUE` charge cannot be circumvented by setuid drop; per-real-uid bookkeeping.
- **GRKERNSEC_AUDIT_IPC** — every `mq_open` with `O_CREAT` audit-logged with uid/euid/pid/exe.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers (no inode addresses leaked).
- **PAX_REFCOUNT** — `dentry`/`inode` refcounts saturating.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — `mq_attr` and any internal `mqueue_inode_info` zeroed on free.
- **GRKERNSEC_RESLOG** — exceeded `RLIMIT_MSGQUEUE` logged per real-uid for forensics.
- **GRKERNSEC_PROC_IPC** — `/proc/sys/fs/mqueue/*` and `/dev/mqueue/*` listings restricted in container/chroot contexts.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `mq_unlink(2)` — separate Tier-5 (`mq_unlink.md`).
- `mq_timedsend(2)` / `mq_timedreceive(2)` — separate Tier-5.
- `mq_notify(2)` / `mq_getsetattr(2)` — separate Tier-5.
- mqueue filesystem internals — covered in `ipc/mqueue.md` Tier-3 if expanded.
- IPC namespace internals — covered in `kernel/nsproxy.md` Tier-3.
- Implementation code.
