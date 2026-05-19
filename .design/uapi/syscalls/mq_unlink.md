# Tier-5: syscall 241 — mq_unlink(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`241  common  mq_unlink  sys_mq_unlink`)
  - ipc/mqueue.c (`SYSCALL_DEFINE1(mq_unlink, ...)`, `do_mq_unlink`, `mqueue_unlink`)
  - fs/namei.c (`filename_lookup`, `mnt_want_write`, `mnt_drop_write`)
  - include/linux/mqueue.h (`struct mqueue_inode_info`)
  - kernel/sysctl.c (`fs.mqueue.queues_max`)
-->

## Summary

`mq_unlink(2)` is **x86_64 syscall 241**, the POSIX message-queue destroyer. Given a queue `name` (in the canonical `/name` form), it removes the queue's directory entry from the IPC-namespace-private `mqueue` filesystem, decrements the queue's accounting against `fs.mqueue.queues_max` and `RLIMIT_MSGQUEUE`, and frees the underlying inode when its final open descriptor closes.

The call is the IPC analogue of `unlink(2)`: the *name* disappears immediately, but the *queue* (and its messages) persists until all `mqd_t` references are closed via `mq_close`/exit, mirroring POSIX shared-memory semantics. Subsequent `mq_open(name, ...)` without `O_CREAT` returns `-ENOENT`; with `O_CREAT|O_EXCL` it succeeds creating a new independent queue. Per IPC-namespace, the operation is constrained to queues created in the caller's `ipc_ns`.

Critical for: cleanup paths after `mq_open`, every container shutdown, every CI integration teardown, every Rookery test that asserts queue lifecycle semantics.

## Signature

C (POSIX / man-pages):

```c
int mq_unlink(const char *name);
```

glibc wrapper: `__mq_unlink` → `INLINE_SYSCALL(mq_unlink, 1, name)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE1(mq_unlink,
                const char __user *, u_name);
```

Rookery dispatch:

```rust
pub fn sys_mq_unlink(
    name: UserPtr<u8>,
) -> SyscallResult<i32>;
```

## Parameters

| name  | type                  | constraints                                                                                       | errno-on-bad           |
|-------|-----------------------|---------------------------------------------------------------------------------------------------|------------------------|
| name  | `const char __user *` | NUL-terminated; must begin with `/`, no further `/`, `1 < strlen ≤ NAME_MAX`.                     | `EFAULT`/`EINVAL`/`ENAMETOOLONG` |

## Return value

- Success: `0`.
- Failure: `-1` with `errno`; in-kernel: a negated errno.

## Errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EFAULT`       | `name` crosses the user/kernel boundary illegally.                                              |
| `EINVAL`       | `name` does not start with `/`, contains additional `/`, is empty, or just `/`.                  |
| `ENAMETOOLONG` | `strlen(name) > NAME_MAX`.                                                                      |
| `ENOENT`       | Named queue does not exist in the caller's IPC namespace.                                       |
| `EACCES`       | Caller lacks write permission on the mqueue directory (i.e., not owner and not `CAP_DAC_OVERRIDE`). |
| `EPERM`        | Sticky bit set on mqueue root and caller is not owner of queue inode and lacks `CAP_FOWNER`.    |
| `EROFS`        | mqueue filesystem mounted read-only (rare; possible inside locked-down containers).             |

## ABI surface (constants + flags)

`mq_unlink(2)` has no flag word; constants of relevance live on the namespace side:

- `NAME_MAX = 255` — per-namespace path-component limit on mqueue filesystem.
- `fs.mqueue.queues_max` — per-IPC-namespace queue count cap; decremented by successful unlink.
- `MS_RDONLY` — mount-flag honored by mqueue superblock if remounted read-only.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=name`.
- REQ-2: `name` must begin with `/`; otherwise `-EINVAL`. A second `/` anywhere ⟹ `-EINVAL`. Empty or lone `/` ⟹ `-EINVAL`.
- REQ-3: `strncpy_from_user(kname, name, NAME_MAX + 1)`; overflow ⟹ `-ENAMETOOLONG`.
- REQ-4: `filename_lookup` issued against the caller's `ipc_ns->mq_mnt.mnt_root`, with `LOOKUP_PARENT` to acquire the parent directory and target dentry.
- REQ-5: `mnt_want_write(mq_mnt)` taken; `mnt_drop_write` on every exit path. Read-only mqueue mount ⟹ `-EROFS`.
- REQ-6: Permission check: caller needs write+execute on mqueue root; if root has sticky bit and caller is not owner of target inode ⟹ `-EPERM`.
- REQ-7: `vfs_unlink(idmap, parent_inode, dentry, NULL)` removes the directory entry; LSM `security_inode_unlink` invoked first.
- REQ-8: Accounting: on successful inode-detach, decrement `ipc_ns->mq_queues_count` and refund `mqueue_inode_info::user.rlimit_msgqueue_charged`.
- REQ-9: Inode is *not* immediately freed: it persists until last `f_count == 0`; outstanding `mqd_t`s on the queue continue to function for send/receive.
- REQ-10: `fsnotify_unlink(dir, dentry)` fired on success.
- REQ-11: Audit hook emitted for POSIX_MQ_UNLINK with name + uid.
- REQ-12: IPC namespace isolation strict: a queue in namespace N1 is invisible from N2.

## Acceptance Criteria

- [ ] AC-1: After `mq_open("/q", O_CREAT, 0600, &a)`, `mq_unlink("/q")` returns 0 and `/dev/mqueue/q` disappears.
- [ ] AC-2: `mq_unlink("/q")` when no queue exists ⟹ `-ENOENT`.
- [ ] AC-3: `mq_unlink("q")` (no leading `/`) ⟹ `-EINVAL`.
- [ ] AC-4: `mq_unlink("/a/b")` ⟹ `-EINVAL`.
- [ ] AC-5: Outstanding `mq_open` fd continues to function after `mq_unlink` until `mq_close`.
- [ ] AC-6: `mq_unlink` + subsequent `mq_open("/q", O_CREAT|O_EXCL, ...)` succeeds (queue name freed).
- [ ] AC-7: Unprivileged unlink of queue owned by other uid (no sticky) ⟹ `-EACCES`.
- [ ] AC-8: With sticky bit on mqueue root, non-owner non-CAP_FOWNER ⟹ `-EPERM`.
- [ ] AC-9: Successful unlink decrements `ipc_ns->mq_queues_count`.
- [ ] AC-10: Successful unlink refunds `RLIMIT_MSGQUEUE` charged bytes to creator's real uid.
- [ ] AC-11: Cross-namespace unlink invisible: queue in N1 not unlinkable from N2.
- [ ] AC-12: `fsnotify_unlink` fired exactly once on success.

## Architecture

```
struct MqUnlinkArgs {
    name: UserPtr<u8>,
}
```

`Mqueue::sys_mq_unlink(args) -> i32`:

1. `let kname = NameBuf::copy_from_user(args.name, NAME_MAX + 1)?;`
2. `if !Mqueue::valid_name(&kname) { return -EINVAL; }`
3. `return Mqueue::do_unlink(&kname);`

`Mqueue::do_unlink(name) -> i32`:

1. `let ns = current_ipc_ns();`
2. `let mnt = ns.mq_mnt.clone();`
3. `mnt_want_write(&mnt)?;`
4. `let (parent, dentry) = filename_lookup(mnt.root, name, LOOKUP_PARENT)?;`
5. `if dentry.is_negative() { mnt_drop_write(&mnt); return -ENOENT; }`
6. `inode_lock_nested(parent.inode, I_MUTEX_PARENT);`
7. `Security::inode_unlink(parent.inode, &dentry)?;`
8. `Mqueue::check_sticky(parent.inode, dentry.inode)?;`
9. `let res = vfs_unlink(mnt.idmap(), parent.inode, &dentry, None);`
10. `if res == 0 { Mqueue::account_unlink(ns, dentry.inode); fsnotify_unlink(parent.inode, &dentry); }`
11. `inode_unlock(parent.inode);`
12. `mnt_drop_write(&mnt);`
13. `return res;`

`Mqueue::account_unlink(ns, inode)`:

1. `let info = mqueue_inode_info(inode);`
2. `atomic_dec(&ns.mq_queues_count);`
3. `Mqueue::uncharge_user(info.user, info.charged_bytes);`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `name_syntax_valid` | INVARIANT | Leading `/`, no interior `/`, non-empty. |
| `mnt_write_paired` | INVARIANT | `mnt_want_write` ↔ `mnt_drop_write` paired on all exits. |
| `sticky_bit_enforced` | INVARIANT | Sticky+non-owner+no-CAP_FOWNER ⟹ `-EPERM`. |
| `accounting_balanced` | INVARIANT | Successful unlink decrements `mq_queues_count` exactly once. |
| `rlimit_refund_balanced` | INVARIANT | Refunded bytes == charged bytes at creation. |
| `ipc_ns_scoped` | INVARIANT | Lookup constrained to caller's IPC ns. |

### Layer 2: TLA+

`uapi/syscalls/mq_unlink.tla`:
- Per-call → name-validate → lookup → permission → vfs_unlink → account.
- Properties:
  - `safety_lookup_within_ipc_ns`,
  - `safety_open_descriptors_survive_unlink`,
  - `safety_account_balanced`,
  - `liveness_mq_unlink_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ name no longer resolvable; existing `mqd_t` still operable | `Mqueue::do_unlink` |
| Post: success ⟹ `mq_queues_count` decremented exactly once | `Mqueue::account_unlink` |
| Post: any error ⟹ on-disk state unchanged, no counters touched | `Mqueue::do_unlink` |
| Post: refunded RLIMIT bytes == originally charged at create | `Mqueue::account_unlink` |

### Layer 4: Verus/Creusot functional

`mq_unlink(name)` ≡ POSIX.1-2017 `mq_unlink` per `man 3 mq_unlink`: the name is removed, the object persists until last close.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mq_unlink(2)` reinforcement:

- **Per-name syntactic gate** — defense against arbitrary-path unlink.
- **Per-`mnt_want_write` discipline** — defense against RO bypass and unmount races.
- **Per-sticky-bit enforcement** — defense against cross-uid queue removal.
- **Per-namespace scoping** — defense against cross-container interference.
- **Per-RLIMIT refund** — defense against accounting drift causing perpetual `-ENOSPC`.
- **Per-`fsnotify_unlink` post-success only** — defense against false events on failure.
- **Per-LSM `inode_unlink` hook** — defense against bypass through alternate VFS entry.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `name` SMAP-guarded; `strncpy_from_user` length-bounded.
- **GRKERNSEC_CHROOT_MQUEUE** — chrooted task cannot unlink queues outside its chroot's IPC namespace.
- **GRKERNSEC_RLIMIT** — `RLIMIT_MSGQUEUE` refund credited to real uid of original creator, not to current effective uid, preventing uid-swap arbitrage.
- **GRKERNSEC_AUDIT_IPC** — every `mq_unlink` (success and failure) logged with uid/exe/name.
- **GRKERNSEC_LINK** — symlink-style abuse on mqueue root rejected.
- **GRKERNSEC_FIFO** — sticky-dir semantics enforced identically to `/tmp`-style unlink.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_REFCOUNT** — `dentry`/`inode`/`mqueue_inode_info` refcounts saturating.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — inode payload zeroed at final free, preventing leak of trailing message bytes.
- **GRKERNSEC_PROC_IPC** — `/dev/mqueue/*` listings restricted to caller's namespace, hiding cross-namespace deletion targets.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `mq_open(2)` — separate Tier-5 (`mq_open.md`).
- `mq_timedsend(2)` / `mq_timedreceive(2)` / `mq_notify(2)` / `mq_getsetattr(2)` — separate Tier-5.
- mqueue filesystem internals — covered in `ipc/mqueue.md` Tier-3 if expanded.
- POSIX shm parallels — covered separately under `mm/shmem.md` Tier-3.
- Implementation code.
