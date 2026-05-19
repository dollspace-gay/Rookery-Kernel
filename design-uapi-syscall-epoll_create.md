---
title: "Tier-5 syscall: epoll_create(2) — syscall 213"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`epoll_create(2)` is the **legacy** single-argument creator for an epoll file descriptor — the "open the event loop" syscall. The `size` argument is a vestige from the 2.6.0 days when epoll used a hash table sized at creation; since 2.6.8 the table is dynamically grown via an RB-tree and `size` is ignored, save for a minimum-positive sanity check (`size <= 0` ⇒ `EINVAL`). The returned fd has neither `O_CLOEXEC` nor `O_NONBLOCK`, mirroring the legacy-fd hazard pattern of `signalfd(2)`, `eventfd(2)`, `inotify_init(2)`. Modern userspace uses `epoll_create1(EPOLL_CLOEXEC)` (syscall 291); the legacy entry remains for ABI stability.

Critical for: server event loops (nginx, haproxy, libuv, libev, tokio-epoll, asio), language runtimes (Node.js, Go netpoll, Java NIO Selector), DBus message-bus daemons, X11 server pollset, container init systems, every "I/O multiplexer" stack on Linux. Hot path for high-fanout I/O.

### Acceptance Criteria

- [ ] AC-1: `epoll_create(1)` returns a valid fd ≥ 0.
- [ ] AC-2: `epoll_create(0)` returns `EINVAL`.
- [ ] AC-3: `epoll_create(-1)` returns `EINVAL`.
- [ ] AC-4: `epoll_create(INT_MAX)` returns a valid fd (size ignored).
- [ ] AC-5: Returned fd has `FD_CLOEXEC == 0` (verified via `fcntl(F_GETFD)`).
- [ ] AC-6: `read(epfd, ...)` returns `EINVAL` (epoll fds are not readable as data).
- [ ] AC-7: `epoll_ctl(epfd, EPOLL_CTL_ADD, epfd, ...)` returns `EINVAL` (self-watch forbidden).
- [ ] AC-8: After per-user instance cap reached, `epoll_create` returns `EMFILE`.
- [ ] AC-9: Closing the last reference frees all watches.
- [ ] AC-10: `epoll_create1(0)` and `epoll_create(1)` produce equivalent fds (modulo fd number).
- [ ] AC-11: After `execve`, legacy epoll fd survives (no `O_CLOEXEC`).
- [ ] AC-12: `dup(epfd)` produces a fd sharing the same RB-tree.

### Architecture

```rust
#[syscall(nr = 213, abi = "sysv")]
pub fn sys_epoll_create(size: i32) -> KResult<i32> {
    /* Legacy: forward to epoll_create1 with flags = 0 after size validation. */
    if size <= 0 { return Err(EINVAL); }
    EpollCreate::do_epoll_create1(0)
}
```

`EpollCreate::do_epoll_create1(flags) -> KResult<i32>`:
1. /* Validate flags */
2. if flags & !EPOLL_CLOEXEC != 0 { return Err(EINVAL); }
3. /* Per-user instance accounting */
4. let user = current().cred.user;
5. let prev = user.epoll_instances.fetch_add(1, Ordering::Relaxed);
6. if prev >= sysctl::epoll_max_user_instances {
   - user.epoll_instances.fetch_sub(1, Ordering::Relaxed);
   - return Err(EMFILE);
7. }
8. /* Allocate eventpoll */
9. let ep = EventPoll::alloc().map_err(|_| ENOMEM)?;
10. /* Anon inode file */
11. let file = anon_inode_getfile("[eventpoll]", &EVENTPOLL_FOPS, ep, O_RDWR)?;
12. /* fd flags from caller */
13. let fd_flags = if flags & EPOLL_CLOEXEC != 0 { O_CLOEXEC } else { 0 };
14. let fd = get_unused_fd_flags(fd_flags)?;
15. fd_install(fd, file);
16. Ok(fd)

`EventPoll::alloc() -> KResult<Arc<EventPoll>>`:
1. let ep = EventPoll {
   - mtx: Mutex::new(()),
   - wq: WaitQueue::new(),
   - poll_wait: WaitQueue::new(),
   - rdllist: ListHead::new(),
   - rbr: RbTree::new(),
   - ovflist: AtomicPtr::null(),
   - user: current().cred.user.clone(),
   - file: AtomicPtr::null(),
   - visited: AtomicU64::new(0),
   - watches: AtomicU64::new(0),
   - mm: current().mm.clone(),
   - busy_poll_usecs: 0,
   - prefer_busy_poll: false,
   - ..
2. }
3. Ok(Arc::new(ep))

`EventPoll::drop()`:   /* file release fop */
1. /* Walk rbr, detach each epitem */
2. for epi in self.rbr.iter() {
   - ep_unregister_pollwait(self, epi);
   - put_file(epi.ffd.file);   /* drop watched-file ref */
3. }
4. /* Per-user instance decrement */
5. self.user.epoll_instances.fetch_sub(1, Ordering::Relaxed);
6. drop(self.user);

### Out of Scope

- `epoll_create1(2)` (Tier-5 separate doc — modern variant with EPOLL_CLOEXEC).
- `epoll_ctl(2)` (Tier-5 separate doc — watch add/mod/del).
- `epoll_wait(2)` (Tier-5 separate doc — event reaping).
- `epoll_pwait(2)` (Tier-5 separate doc — sigmask variant).
- RB-tree implementation (Tier-3 in `lib/rbtree.md`).
- Implementation code.

### signature

```c
int epoll_create(int size);
```

### parameters

| Param | Type | Direction | Semantics |
|---|---|---|---|
| `size` | `int` | in | Hint of expected fd count. **Ignored** since 2.6.8 except for the validity check `size > 0`. |

### return value

| Value | Meaning |
|---|---|
| ≥ 0 | New epoll file descriptor. |
| `-1` | Error; `errno` set. |

### errors

| `errno` | Cause |
|---|---|
| `EINVAL` | `size <= 0`. |
| `EMFILE` | Per-process fd limit reached; or per-user `/proc/sys/fs/epoll/max_user_instances` exceeded. |
| `ENFILE` | System-wide fd table exhausted. |
| `ENOMEM` | Kernel could not allocate `struct eventpoll`. |

### abi surface

```text
__NR_epoll_create  (x86_64) = 213
__NR_epoll_create  (i386)   = 254
__NR_epoll_create  (arm64)  = NOT IMPLEMENTED   (arm64 only provides epoll_create1)
__NR_epoll_create  (generic)= NOT IMPLEMENTED

/* Returned fd is an anon_inode "[eventpoll]" file. */

struct epoll_event {
    __poll_t   events;   /* EPOLLIN | EPOLLOUT | EPOLLET | ... */
    epoll_data_t data;   /* opaque user-cookie: ptr / fd / u32 / u64 */
} __attribute__((__packed__));   /* on x86_64 only; aligned on other arches */

/* Per-user resource cap (sysctl-tunable): */
fs.epoll.max_user_instances = 128   /* default per-uid */
fs.epoll.max_user_watches   = computed-from-RAM
```

### compatibility contract

REQ-1: Syscall number is **213** on x86_64; **254** on i386; **NOT exposed** on arm64/generic (modern arches only provide `epoll_create1`). Kernel uses `__ARCH_WANT_SYS_EPOLL_CREATE` to compile-in legacy entry.

REQ-2: `size <= 0` ⇒ `EINVAL`. `size > 0` is otherwise ignored; the kernel never actually allocates `size` slots. Userspace MUST NOT rely on the value (`epoll_create(1)` is idiomatic).

REQ-3: Returned fd has flags `0`: NO `O_CLOEXEC`, NO `O_NONBLOCK`. Survives `execve` unless `FD_CLOEXEC` is later set.

REQ-4: Returned fd refers to an anonymous inode "[eventpoll]" backed by `struct eventpoll`. `dup(2)`/`dup2(2)`/`fcntl(F_DUPFD)` work; all share underlying RB-tree of watches.

REQ-5: Per-user instance cap: `fs.epoll.max_user_instances` (default 128). Exceeded ⇒ `EMFILE`. Accounted via `user_struct::epoll_instances` atomic.

REQ-6: Per-user watch cap: `fs.epoll.max_user_watches` (auto-tuned from RAM, ~4% of LowMem). Accounted via `user_struct::epoll_watches` atomic; charged on `epoll_ctl(EPOLL_CTL_ADD)`.

REQ-7: `struct eventpoll` lifetime: refcount via the backing file; the last `close(2)` on any dup releases the struct, walks the RB-tree, and detaches all watches.

REQ-8: Closing an epoll fd that has watches: the kernel walks `ep->rbr`, removes each entry, drops the watched-file references, and frees. Per-watch cleanup is O(N) at close.

REQ-9: An epoll fd watching ITSELF is forbidden: `epoll_ctl(epfd, EPOLL_CTL_ADD, epfd, ...)` ⇒ `EINVAL`. Cycles between epoll fds are detected via DFS to depth `EP_MAX_NESTS = 4`; deeper ⇒ `ELOOP`.

REQ-10: `epoll_create(2)` and `epoll_create1(2)` are functionally equivalent for `flags = 0`; the difference is the legacy `size` argument acceptance and the `EPOLL_CLOEXEC` ability.

REQ-11: LSM hook `security_file_alloc` fires on creation (anon_inode file).

REQ-12: Per-PID-namespace: epoll fd is per-task and not ns-aware in itself; the watched fds may be from any ns the task can see.

REQ-13: Memory accounting: `struct eventpoll` (~256 bytes) charged to caller's memcg; per-watch `epitem` (~144 bytes) charged on `EPOLL_CTL_ADD`.

REQ-14: `seccomp` filters see `size` verbatim; SHOULD be unrestricted but MAY be blocked in favor of `epoll_create1`.

REQ-15: NOT signal-safe; `epoll_create` allocates and takes mutexes. Async-signal-safety NOT mandated by POSIX (this is Linux-specific).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `size_positive` | INVARIANT | per-epoll_create: size > 0 mandatory. |
| `instance_cap_enforced` | INVARIANT | per-create: user.epoll_instances ≤ sysctl max. |
| `legacy_no_cloexec` | INVARIANT | per-legacy: returned fd has FD_CLOEXEC unset. |
| `refcount_balanced` | INVARIANT | per-create/close: file refcount and ep.user balanced. |
| `self_watch_forbidden` | INVARIANT | per-EPOLL_CTL_ADD: target != epfd. |
| `nesting_depth_bounded` | INVARIANT | per-cycle-check: depth ≤ EP_MAX_NESTS. |

### Layer 2: TLA+

`fs/epoll-create.tla`:
- States: per-user epoll_instances counter, per-fd RB-tree of epitems.
- Properties:
  - `safety_instance_cap` — per-user: epoll_instances never exceeds sysctl max.
  - `safety_no_leak_on_emfile` — per-EMFILE: instance counter decremented.
  - `liveness_create_succeeds_under_cap` — per-create: with quota, returns a fd.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_epoll_create1` post: ret fd corresponds to anon_inode[eventpoll] | `EpollCreate::do_epoll_create1` |
| `alloc` post: rbr empty, watches == 0 | `EventPoll::alloc` |
| `drop` post: all watches released, user counter decremented | `EventPoll::drop` |

### Layer 4: Verus / Creusot functional

Per-`epoll_create(2)` man-page equivalence. LTP `epoll_create01..02` pass. `epoll_create(1)` and `epoll_create1(0)` produce functionally identical epoll fds.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`epoll_create(2)` legacy reinforcement:

- **Per-size validity gate** — defense against per-zero/negative-hint corner-case.
- **Per-user instance cap** — defense against per-user epoll DoS.
- **Per-cycle detection via DFS** — defense against per-cyclic-epoll infinite-loop.
- **Per-RB-tree integrity** — defense against per-corrupt-watch UAF.
- **Per-anon_inode anonymity** — defense against per-procfs lookup of epoll internals.

### grsecurity / pax-style reinforcement

- **EPOLL fd-lifetime refcount strict** — grsec hardens `ep_free` to verify that `rbr` is fully empty before kfree; any leaked epitem is caught via WARN_ON+leak-detector (panic if `panic_on_oom`).
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/fdinfo/<epfd>` watch list is filtered for non-owners; only the epoll-fd's owner can enumerate watches.
- **GRKERNSEC_HARDEN_PTRACE process-isolation** — peer processes cannot inspect another's epoll watches via ptrace without CAP_SYS_PTRACE.
- **signalfd CLOEXEC mandatory analog** — grsec REWRITES legacy `epoll_create(2)` to forcibly set `FD_CLOEXEC`, eliminating post-execve epoll-leak. Userspace that depends on the legacy "fd survives execve" semantics MUST use `epoll_create1(EPOLL_CLOEXEC)` explicitly to opt-out (sysctl `kernel.epoll_legacy_cloexec_default = 1`).
- **PIDFD_SELF / pidfd interplay** — epoll is per-task by file refcount; watching `pidfd` for process-exit is supported, but grsec ensures `pidfd_open(PIDFD_SELF)` watches do not create unbreakable cycles.
- **PaX UDEREF on epoll_ctl payload** — not applicable to epoll_create itself (no userspace pointer arg); the `epoll_event` UDEREF check happens in `epoll_ctl`/`epoll_wait`.
- **GRKERNSEC_AUDIT_GROUP** — burst-creation of epoll fds (epoll-instance reconnaissance / DoS) from a marked group is rate-limited / audited.
- **No_new_privs neutral** — NNP does not change epoll semantics.
- **Anti-fingerprint hardening** — `epoll_create` does not expose any kernel-pointer or address-space layout; the returned fd is opaque.
- **Per-uid quota strict** — `epoll_max_user_instances` cannot be raised without CAP_SYS_ADMIN; cgroup-v2 io.max may further constrain.
- **Seccomp interaction** — seccomp filters may explicitly block syscall 213 in favor of 291; default-deny hardening recommends forbidding legacy epoll_create entirely.
- **Capability gate for cross-task watching** — epoll watching of fds belonging to other processes (via /proc/<pid>/fd/<n>) requires the same access controls as opening that fd directly; grsec reinforces that watching does not bypass open-time LSM.

