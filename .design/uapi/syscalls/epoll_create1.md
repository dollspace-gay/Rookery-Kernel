# Tier-5 syscall: epoll_create1(2) — syscall 291

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/eventpoll.c (sys_epoll_create1, do_epoll_create)
  - include/uapi/linux/eventpoll.h
  - arch/*/include/generated/uapi/asm/unistd_64.h (291)
  - Documentation/admin-guide/epoll.rst, man epoll_create1(2), man epoll(7)
-->

## Summary

`epoll_create1(2)` allocates a new epoll instance — a kernel object that holds an "interest list" of file descriptors plus a "ready list" of fds whose I/O state has changed — and returns a file descriptor that subsequent `epoll_ctl(2)` and `epoll_wait`/`epoll_pwait`/`epoll_pwait2(2)` syscalls operate on. Replaces the legacy `epoll_create(2)` (syscall 213), which took a (now-ignored) `size` hint; `epoll_create1` accepts a flags argument so userspace can request `O_CLOEXEC`. Critical for: every modern event-loop framework (libuv, libevent, libev, tokio MIO, asyncio, golang netpoller, nginx, redis, haproxy, envoy, postgres, mariadb), edge-triggered TCP reactors, and large-fd-count servers.

This Tier-5 covers the userspace ABI of syscall 291.

## Signature

```c
int epoll_create1(int flags);
```

Rust ABI shim:

```rust
pub fn sys_epoll_create1(flags: i32) -> isize;
```

Syscall number: **291**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `flags` | `i32` | IN | `0` or `EPOLL_CLOEXEC` (`O_CLOEXEC`); any other bit ⟹ `-EINVAL` |

## Return

- **Success**: non-negative `int` file descriptor referencing the new epoll instance. The fd is `O_RDWR`-equivalent for kernel operations, accepts `epoll_ctl` / `epoll_wait` / `epoll_pwait` / `epoll_pwait2`, supports `close(2)`, `dup(2)`, `fcntl(F_GETFD/F_SETFD)`, and can itself be added to another epoll's interest list (epoll-on-epoll nesting up to `EP_MAX_NESTS = 4`).
- **Failure**: `-1` with `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `flags` contains a bit other than `EPOLL_CLOEXEC` |
| `EMFILE` | per-process fd table full (`RLIMIT_NOFILE`); or per-user-watch limit (`/proc/sys/fs/epoll/max_user_watches` reached when probe-allocating) |
| `ENFILE` | system-wide fd table full |
| `ENOMEM` | kernel cannot allocate `struct eventpoll` or red-black tree backing |
| `EPERM` | per-namespace policy blocks epoll creation (rare, under restrictive LSM) |

## ABI surface

`EPOLL_*` flags:

| Constant | Value | Purpose |
|---|---|---|
| `EPOLL_CLOEXEC` | `O_CLOEXEC` (`0x80000` / `02000000` octal on most arches) | close-on-exec |

Note: `EPOLL_NONBLOCK` does not exist as a creation flag — non-blocking semantics for `epoll_wait` are controlled by the `timeout` argument (0 = poll, -1 = block forever).

Legacy companion `epoll_create(2)` (syscall 213):
- Signature: `int epoll_create(int size);`.
- `size` argument is ignored since 2.6.8 (was previously a hint).
- Behavior identical to `epoll_create1(0)` for current kernels.
- Provided for ABI compatibility only.

## Compatibility contract

REQ-1: `flags` validation:
- `flags & ~EPOLL_CLOEXEC != 0` → `-EINVAL`.
- `flags == 0` permitted (fd inherits across exec).
- `flags == EPOLL_CLOEXEC` permitted (fd closed on exec).

REQ-2: fd properties:
- Returned fd is non-negative.
- Accepts `read(2)` / `write(2)` ⟹ `-EINVAL` (epoll fds are not data fds).
- `fstat(2)` returns `S_ISFIFO`-style anon-inode metadata.
- `fcntl(F_GETFL)` returns `O_RDWR` (not user-modifiable to e.g. add `O_NONBLOCK` semantics).

REQ-3: Interest list:
- Initially empty.
- Manipulated only via `epoll_ctl(2)`.

REQ-4: Per-user watch limit:
- Each item added (later) charged against `/proc/sys/fs/epoll/max_user_watches`.
- Creation itself does not pre-allocate watches — `max_user_watches` applies on `EPOLL_CTL_ADD`.

REQ-5: Per-process fd accounting:
- Returned fd consumes one `RLIMIT_NOFILE` slot.

REQ-6: Lifetime:
- `close(2)` triggers full teardown: remove every fd from interest list (decrementing target's epitem list), free RB-tree, free ready-list, drop `struct eventpoll` refcount.
- `dup(2)` / `fork(2)`: shared `struct file`; both holders see same eventpoll.

REQ-7: Epoll-on-epoll:
- An epoll fd may be added to another epoll's interest list.
- Cycle detection enforced at `epoll_ctl(EPOLL_CTL_ADD)` time (per `EP_MAX_NESTS = 4`).
- Creation does not validate nesting.

REQ-8: anon_inode:
- Underlying inode is anon_inode with `[eventpoll]` name (visible in `/proc/<pid>/fd/<N>` symlink).
- Kernel may also expose `/proc/<pid>/fdinfo/<N>` with `tfd:` entries describing interest list (for `EPOLL` debugging tooling).

REQ-9: O_CLOEXEC equivalence:
- `EPOLL_CLOEXEC` is bit-equivalent to `O_CLOEXEC`.
- May be queried/cleared via `fcntl(fd, F_GETFD/F_SETFD, FD_CLOEXEC)`.

REQ-10: No size hint:
- Unlike legacy `epoll_create(2)`, `epoll_create1` takes no size hint.
- RB-tree grows dynamically; size is not pre-allocated.

## Acceptance Criteria

- [ ] AC-1: `epoll_create1(0)` returns fd ≥ 0; fd is anon-inode epoll instance.
- [ ] AC-2: `epoll_create1(EPOLL_CLOEXEC)` returns fd; `fcntl(fd, F_GETFD)` has `FD_CLOEXEC` set.
- [ ] AC-3: `epoll_create1(0x1)` (any non-CLOEXEC bit) → `-EINVAL`.
- [ ] AC-4: `epoll_create1(0xffffffff)` → `-EINVAL`.
- [ ] AC-5: Created fd: `read(fd, buf, 1)` → `-EINVAL`.
- [ ] AC-6: Created fd: `write(fd, buf, 1)` → `-EINVAL`.
- [ ] AC-7: Created fd: usable in `epoll_ctl(other, ADD, fd, ...)` (epoll-on-epoll), respecting cycle/nest limit.
- [ ] AC-8: `RLIMIT_NOFILE` at limit → `-EMFILE`.
- [ ] AC-9: Created fd: `close(fd)` succeeds and frees `struct eventpoll`.
- [ ] AC-10: Created fd: `dup(fd)` returns second fd; closing one does not free until both closed.
- [ ] AC-11: After `execve` with `EPOLL_CLOEXEC` set: fd absent in new image.
- [ ] AC-12: After `execve` without `EPOLL_CLOEXEC`: fd present and operational in new image.
- [ ] AC-13: `epoll_create(0)` (legacy syscall 213) behaves identically to `epoll_create1(0)`.
- [ ] AC-14: Created fd appears in `/proc/<pid>/fd/` as `anon_inode:[eventpoll]`.

## Architecture

Rookery surface in `kernel/eventpoll/create.rs`:

```rust
bitflags! {
    pub struct EpollCreateFlags: i32 {
        const CLOEXEC = 0x80000;     /* equiv O_CLOEXEC */
    }
}

pub struct EventPollCtx {
    pub rb_tree:   RwLock<RBTree<EpollKey, Arc<EpItem>>>,
    pub rdllist:   Mutex<LinkedList<Arc<EpItem>>>,
    pub ovflist:   Mutex<Option<NonNull<EpItem>>>,    /* overflow list during scan */
    pub wq:        WaitQueue,
    pub poll_wait: WaitQueue,
    pub user:      Arc<UserNamespace>,
    pub watches:   AtomicUsize,
    pub nests:     u32,                                /* for epoll-on-epoll */
}
```

`EpollCreate::create1(flags) -> isize`:
1. /* flag validation */
2. if (flags & !EpollCreateFlags::CLOEXEC.bits()) != 0 { return -EINVAL; }
3. /* per-user watch limit pre-check (informational) */
4. let user = current.user_ns.clone();
5. if user.epoll_watches_count() >= sysctl.fs_epoll_max_user_watches() {
     - /* note: creation succeeds; pre-existing watch saturation surfaces on first CTL_ADD */
   }
6. /* allocate ctx */
7. let ctx = Arc::new(EventPollCtx {
     - rb_tree: RwLock::new(RBTree::new()),
     - rdllist: Mutex::new(LinkedList::new()),
     - ovflist: Mutex::new(None),
     - wq: WaitQueue::new(),
     - poll_wait: WaitQueue::new(),
     - user,
     - watches: AtomicUsize::new(0),
     - nests: 0,
   });
8. /* anon_inode fd */
9. let open_flags = if (flags & EpollCreateFlags::CLOEXEC.bits()) != 0 { O_RDWR | O_CLOEXEC } else { O_RDWR };
10. let fd = anon_inode_getfd("[eventpoll]", &EVENTPOLL_FOPS, ctx, open_flags)?;
11. return fd as isize;

`EpollCreate::create_legacy(size_ignored) -> isize`:
1. /* size argument silently ignored since 2.6.8 */
2. return EpollCreate::create1(0);

`EVENTPOLL_FOPS` rejects userspace `read` / `write`:
1. read  ⟹ -EINVAL.
2. write ⟹ -EINVAL.
3. poll  ⟹ honored — supports epoll-on-epoll.
4. release ⟹ teardown rb_tree + rdllist + decrement target epitems.
5. ioctl ⟹ EPIOCSPARAMS / EPIOCGPARAMS (timer-slack, busy-poll knobs).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_subset` | INVARIANT | `flags & ~CLOEXEC != 0 ⟹ -EINVAL` |
| `cloexec_propagates` | INVARIANT | `CLOEXEC ⟹ FD_CLOEXEC set` |
| `data_ops_rejected` | INVARIANT | read/write on epoll fd ⟹ -EINVAL |
| `fd_install_atomic` | INVARIANT | fd in fdtable iff ctx fully initialized |
| `watches_zero_at_create` | INVARIANT | ctx.watches == 0 post-create |

### Layer 2: TLA+

`uapi/epoll_create1.tla`:
- States: `validating`, `allocating`, `installing_fd`, `returned`, `failed`.
- Properties:
  - `safety_no_partial_install` — fd visible iff ctx fully initialized.
  - `safety_cloexec_consistent` — fd's FD_CLOEXEC reflects flags arg.
  - `liveness_terminate` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `create1` post(ok): `0 ≤ ret` ∧ ctx.rb_tree empty | `EpollCreate::create1` |
| `create1` post: ctx.watches == 0 | `EpollCreate::create1` |
| `create_legacy` post: equivalent to `create1(0)` | `EpollCreate::create_legacy` |

### Layer 4: Verus/Creusot functional

Per-`epoll_create1(2)` man page, `man epoll(7)`, `Documentation/admin-guide/epoll.rst` semantic equivalence. Per-LTP `testcases/kernel/syscalls/epoll_create1/*.c` and glibc `sysdeps/unix/sysv/linux/epoll_create1.c` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

epoll_create1 reinforcement:

- **CLOEXEC default under restrictive policy** — defense against per-exec leak of epoll fd to setuid child.
- **Per-user-namespace watch limit honored** — defense against per-user watch exhaustion.
- **Read/write strict rejection on epoll fd** — defense against per-fd-confusion exploit.
- **anon_inode source isolation** — defense against per-namespace cross-leak.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on epoll_create1 entry** — randomize kernel stack; though epoll_create1 is low-frequency relative to epoll_wait, it is the entry that establishes the `struct eventpoll` allocation footprint and thus a known target for SLAB-grooming primitives.
- **PaX UDEREF not directly applicable** — the syscall takes only an `int` flags arg; no user pointers to validate. UDEREF inherited from generic syscall entry.
- **GRKERNSEC_BPF_HARDEN not directly applicable** — epoll is not a programmable VM. However the *interest list* manipulated by sibling `epoll_ctl` is a user-controlled kernel-resident table, and analogous hardening applies there (see `epoll_ctl.md`).
- **CAP_BPF gating not appropriate** — epoll is a baseline userspace primitive. Apply per-uid quota instead.
- **GRKERNSEC_FIFO scaled to epoll-create flood** — per-uid quota on simultaneous open epoll instances per uid; refuses creation with `-EMFILE` past quota; breaks the "create N million epoll instances to exhaust SLAB" DoS class. Mirrors grsec's GRKERNSEC_FIFO doctrine for pipes.
- **EPOLL_CLOEXEC mandatory under restrictive policy** — when calling task has `PR_SET_NO_NEW_PRIVS=1`, runs under crosslink seccomp lockdown, or is invoked across a setuid boundary, refuse `epoll_create1` without `EPOLL_CLOEXEC`; eliminates a cross-exec covert channel where parent leaks epoll instance with active watches to a less-privileged child.
- **GRKERNSEC_HIDESYM on epoll fdinfo** — `/proc/<pid>/fdinfo/<N>` for epoll fds enumerates per-watch `tfd:` entries which leak observed fd values across the calling task's fd namespace; under `kernel.kptr_restrict ≥ 2`, fdinfo is restricted to `CAP_SYS_ADMIN` and same-uid + same-pidns observers.
- **Per-user max_user_watches enforcement (defense-in-depth)** — even with `RLIMIT_NOFILE` permitting creation, refuse `epoll_create1` when the calling user's existing watch count already exceeds `max_user_watches` × policy-coefficient; defense against per-user pre-creation watch exhaustion.
- **anon_inode label propagation** — the new epoll inode inherits the calling task's SELinux / AppArmor label; refuse cross-domain `add` later (enforced in `epoll_ctl`).
- **Audit log on epoll_create1 from unprivileged + suid-history task** — task with `dumpable == 0` (post-setuid) creating epoll instance is logged at `LOGLEVEL_INFO` for forensic correlation; common pre-step in fd-confusion exploits.
- **Strict reject of legacy `epoll_create(size)` with `size > 1<<20`** — even though size is ignored, refuse implausibly large values to break primitive-probing.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `epoll_ctl.md` sibling — interest list manipulation
- `epoll_wait.md` / `epoll_pwait2.md` siblings — readiness retrieval
- `fs/eventpoll.md` Tier-3 — rb_tree / ready-list / wakeup-callback mechanics
- `epoll_create(2)` legacy syscall 213 — captured here as compatibility shim
- Implementation code
