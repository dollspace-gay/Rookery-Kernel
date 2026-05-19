# Tier-5 syscall: epoll_ctl(2) — syscall 233

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/eventpoll.c (sys_epoll_ctl, do_epoll_ctl)
  - include/uapi/linux/eventpoll.h
  - arch/*/include/generated/uapi/asm/unistd_64.h (233)
  - Documentation/admin-guide/epoll.rst, man epoll_ctl(2), man epoll(7)
-->

## Summary

`epoll_ctl(2)` mutates the **interest list** of an epoll instance. The syscall multiplexes three operations — `EPOLL_CTL_ADD`, `EPOLL_CTL_MOD`, `EPOLL_CTL_DEL` — that add a new fd to the interest list, modify its event mask and user data, or remove it. Each interest-list entry is a kernel `struct epitem` keyed by `(file, fd)` and stored in a per-instance red-black tree; entries that observe an event are linked into the per-instance ready list and visible to `epoll_wait(2)` / `epoll_pwait(2)` / `epoll_pwait2(2)`. Critical for: every userspace event loop's "register/modify/unregister fd" operation; level- vs edge-triggered semantics; one-shot semantics; exclusive (`EPOLLEXCLUSIVE`) wakeup to defeat thundering herd in multi-acceptor servers.

This Tier-5 covers the userspace ABI of syscall 233.

## Signature

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

Rust ABI shim:

```rust
pub fn sys_epoll_ctl(epfd: i32,
                     op: i32,
                     fd: i32,
                     event: *mut EpollEvent) -> isize;
```

Syscall number: **233**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `epfd` | `i32` | IN | epoll instance fd from `epoll_create1(2)` |
| `op` | `i32` | IN | `EPOLL_CTL_ADD` / `EPOLL_CTL_MOD` / `EPOLL_CTL_DEL` |
| `fd` | `i32` | IN | target fd to add / modify / remove |
| `event` | `struct epoll_event *` | IN | event mask + user data (NULL permitted only for `DEL`) |

`struct epoll_event` (UAPI, **packed on x86 only**; aligned on other arches):

```c
struct epoll_event {
    __poll_t events;       /* EPOLL* event mask (32-bit) */
    epoll_data_t data;     /* opaque 64-bit user data    */
} __EPOLL_PACKED;

typedef union epoll_data {
    void    *ptr;
    int      fd;
    __u32    u32;
    __u64    u64;
} epoll_data_t;
```

Total size: **12 bytes** on x86 (packed), **16 bytes** on other arches.

## Return

- **Success**: `0`.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `epfd` or `fd` not a valid fd |
| `EEXIST` | `EPOLL_CTL_ADD` and `(file, fd)` already in interest list |
| `EINVAL` | `epfd == fd`; unknown `op`; per-op-invalid mask (e.g., `EPOLLEXCLUSIVE` on `MOD`); target fd does not support `poll`; epoll-on-epoll cycle would be created; `EPOLL_CTL_DEL` does not accept `EPOLLEXCLUSIVE` |
| `ENOENT` | `EPOLL_CTL_MOD` or `EPOLL_CTL_DEL` and `(file, fd)` not in interest list |
| `ENOMEM` | per-`struct epitem` allocation failed |
| `ENOSPC` | per-user-namespace `max_user_watches` exceeded on `ADD` |
| `EPERM` | target fd's file does not implement `poll()` (e.g., regular file); or LSM denies attach |
| `ELOOP` | adding `fd` would create a cycle in nested epoll graph, or nest-depth would exceed `EP_MAX_NESTS = 4` |
| `EFAULT` | `event` not in user-space-readable address range (when required) |

## ABI surface

`EPOLL_CTL_*` operations:

| Constant | Value | Purpose |
|---|---|---|
| `EPOLL_CTL_ADD` | `1` | add (file, fd) to interest list |
| `EPOLL_CTL_DEL` | `2` | remove (file, fd) from interest list |
| `EPOLL_CTL_MOD` | `3` | replace event mask + data for (file, fd) |

`EPOLL*` event bits (low 16 bits track `POLL*` ABI; high bits epoll-specific):

| Constant | Value | Purpose |
|---|---|---|
| `EPOLLIN` | `0x001` | read possible |
| `EPOLLPRI` | `0x002` | urgent data available |
| `EPOLLOUT` | `0x004` | write possible |
| `EPOLLERR` | `0x008` | error condition (always implicitly reported) |
| `EPOLLHUP` | `0x010` | hang-up (always implicitly reported) |
| `EPOLLRDNORM` | `0x040` | normal data (TCP) |
| `EPOLLRDBAND` | `0x080` | priority band data |
| `EPOLLWRNORM` | `0x100` | normal write (TCP) |
| `EPOLLWRBAND` | `0x200` | priority band write |
| `EPOLLMSG` | `0x400` | (unused) |
| `EPOLLRDHUP` | `0x2000` | peer closed write side |
| `EPOLLEXCLUSIVE` | `1 << 28` | one waiter wakes (mitigate thundering herd) |
| `EPOLLWAKEUP` | `1 << 29` | inhibit suspend while event pending |
| `EPOLLONESHOT` | `1 << 30` | disarm after first event |
| `EPOLLET` | `1 << 31` | edge-triggered |

Mutual exclusion rules:

- `EPOLLEXCLUSIVE` valid only for `EPOLL_CTL_ADD`; not for `MOD` (modify path cannot promote/demote exclusivity); not for `DEL`.
- `EPOLLEXCLUSIVE` mutually exclusive with `EPOLLONESHOT`.
- `EPOLLWAKEUP` requires `CAP_BLOCK_SUSPEND`.
- `EPOLLEXCLUSIVE` cannot be set when target is itself an epoll fd.

## Compatibility contract

REQ-1: `op` validation:
- Must be `EPOLL_CTL_ADD`/`MOD`/`DEL` else `-EINVAL`.

REQ-2: fd validation:
- `epfd` must be open and be an epoll instance; else `-EBADF` (not-open) or `-EINVAL` (open but not epoll).
- `fd` must be open; else `-EBADF`.
- `epfd == fd` → `-EINVAL`.

REQ-3: Target fd poll support:
- Target's `struct file_operations.poll` must be non-NULL; else `-EPERM`.
- Regular files do not support epoll (no `.poll`); chrdev/sock/pipe/eventfd/timerfd/signalfd/epoll/io_uring all do.

REQ-4: `EPOLL_CTL_ADD`:
- `event != NULL` required; `NULL` ⟹ `-EFAULT`.
- Copy `event` from user.
- Check uniqueness via RB-tree lookup by `(file, fd)` key; duplicate ⟹ `-EEXIST`.
- Cycle / nest check if target is an epoll fd: traverse to `EP_MAX_NESTS = 4`; over ⟹ `-ELOOP`.
- Allocate `struct epitem`; charge `max_user_watches`.
- Register wait-queue entry on target's `poll_waitqueue` via `ep_ptable_queue_proc`.
- Insert into RB-tree.
- If target is currently ready, queue to ready list.

REQ-5: `EPOLL_CTL_MOD`:
- `event != NULL` required.
- Locate `(file, fd)` in RB-tree; absent ⟹ `-ENOENT`.
- Reject `EPOLLEXCLUSIVE` modification ⟹ `-EINVAL`.
- Atomically replace `event.events` (preserving implicit `EPOLLERR | EPOLLHUP`) and `event.data`.
- If poll state matches new mask, queue to ready list.

REQ-6: `EPOLL_CTL_DEL`:
- `event` ignored (may be `NULL`).
- Locate `(file, fd)` in RB-tree; absent ⟹ `-ENOENT`.
- Unregister wait-queue entry from target's `poll_waitqueue`.
- Remove from ready list if present.
- Remove from RB-tree.
- Free `struct epitem`; uncharge `max_user_watches`.

REQ-7: Implicit events:
- `EPOLLERR` and `EPOLLHUP` always reported, even if not in `events` mask.

REQ-8: Edge-triggered (`EPOLLET`):
- Event reported only on transition to ready.
- Userspace must drain target until `-EAGAIN` to re-arm reporting.

REQ-9: One-shot (`EPOLLONESHOT`):
- After first wake, mask cleared internally (entry remains, mask zeroed except errors).
- Re-arm requires `EPOLL_CTL_MOD`.

REQ-10: Exclusive (`EPOLLEXCLUSIVE`):
- Multiple epoll instances attached to same fd: only one is woken per event.
- Mitigates listening-socket thundering herd.

REQ-11: Cycle / nest check:
- Adding epoll fd `B` to epoll fd `A`'s interest list checks `B → ... → A` reachability and depth bound.
- Bounds: `EP_MAX_NESTS = 4`.

REQ-12: Per-user watch limit:
- `ADD` checks `current_user.epoll_watches < /proc/sys/fs/epoll/max_user_watches`; over ⟹ `-ENOSPC`.
- Default cap: 4% of low memory, ~96 MiB worth of `struct epitem` total per user.

REQ-13: Concurrency:
- `epoll_ctl` is fully reentrant per epoll instance via per-instance mutex.
- Concurrent `epoll_wait` proceeds; new entries added during scan are visible in next scan; entries removed are skipped.

REQ-14: `EPOLLWAKEUP` capability:
- Requires `CAP_BLOCK_SUSPEND`; else flag silently cleared (no error).

REQ-15: Target file lifetime:
- `struct epitem` holds `fget`-reference to target file; target survives until `EPOLL_CTL_DEL` or epoll teardown.

## Acceptance Criteria

- [ ] AC-1: `epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev)` with valid pipe fd → 0.
- [ ] AC-2: Second `ADD` of same fd → `-EEXIST`.
- [ ] AC-3: `MOD` on non-existent fd → `-ENOENT`.
- [ ] AC-4: `DEL` on non-existent fd → `-ENOENT`.
- [ ] AC-5: `ADD` of regular file fd → `-EPERM`.
- [ ] AC-6: `ADD` with `epfd == fd` → `-EINVAL`.
- [ ] AC-7: `op = 99` → `-EINVAL`.
- [ ] AC-8: `event == NULL` on `ADD` → `-EFAULT`.
- [ ] AC-9: `event == NULL` on `DEL` → 0 (ignored).
- [ ] AC-10: `ADD` with `EPOLLEXCLUSIVE | EPOLLONESHOT` → `-EINVAL`.
- [ ] AC-11: `MOD` with `EPOLLEXCLUSIVE` → `-EINVAL`.
- [ ] AC-12: `ADD` of epoll fd creating depth > 4 → `-ELOOP`.
- [ ] AC-13: Implicit `EPOLLERR | EPOLLHUP` reported even without explicit subscription.
- [ ] AC-14: Per-user `max_user_watches` exhausted → `-ENOSPC`.
- [ ] AC-15: `EPOLLWAKEUP` without `CAP_BLOCK_SUSPEND` → silently cleared.

## Architecture

Rookery surface in `kernel/eventpoll/ctl.rs`:

```rust
#[repr(i32)]
pub enum EpollCtlOp {
    Add = 1,
    Del = 2,
    Mod = 3,
}

#[repr(C, packed(4))]   /* x86 layout; aligned on others via per-arch cfg */
pub struct EpollEvent {
    pub events: u32,
    pub data:   u64,
}

bitflags! {
    pub struct EpollEvents: u32 {
        const IN         = 0x001;
        const PRI        = 0x002;
        const OUT        = 0x004;
        const ERR        = 0x008;
        const HUP        = 0x010;
        const RDNORM     = 0x040;
        const RDBAND     = 0x080;
        const WRNORM     = 0x100;
        const WRBAND     = 0x200;
        const MSG        = 0x400;
        const RDHUP      = 0x2000;
        const EXCLUSIVE  = 1 << 28;
        const WAKEUP     = 1 << 29;
        const ONESHOT    = 1 << 30;
        const ET         = 1 << 31;
    }
}

pub const EP_MAX_NESTS: u32 = 4;

pub struct EpItem {
    pub rb_node:    RBNode,
    pub key:        EpollKey,        /* (file, fd) */
    pub ep:         Arc<EventPollCtx>,
    pub event:      RwLock<EpollEvent>,
    pub waiter:     PollWaitEntry,
    pub ready_link: ListNode,
}
```

`Epoll::ctl(epfd, op, fd, ev_uptr) -> isize`:
1. /* resolve epoll ctx */
2. let ep_file = current.fdtable.get(epfd).ok_or(-EBADF)?;
3. let ctx = ep_file.as_eventpoll().ok_or(-EINVAL)?;
4. /* resolve target file */
5. let tgt_file = current.fdtable.get(fd).ok_or(-EBADF)?;
6. /* self check */
7. if epfd == fd { return -EINVAL; }
8. /* poll support */
9. if tgt_file.fops.poll.is_none() { return -EPERM; }
10. /* op dispatch */
11. let op = EpollCtlOp::try_from(op).map_err(|_| -EINVAL)?;
12. match op {
    - Add => Epoll::add(&ctx, fd, &tgt_file, ev_uptr),
    - Mod => Epoll::modify(&ctx, fd, &tgt_file, ev_uptr),
    - Del => Epoll::remove(&ctx, fd, &tgt_file),
   }

`Epoll::add(ctx, fd, tgt_file, ev_uptr)`:
1. /* event required */
2. if ev_uptr.is_null() { return Err(-EFAULT); }
3. let mut ev = copy_from_user::<EpollEvent>(ev_uptr).map_err(|_| -EFAULT)?;
4. /* sanitize flags */
5. let events = EpollEvents::from_bits(ev.events).ok_or(-EINVAL)?;
6. if events.contains(EpollEvents::EXCLUSIVE) {
     - if events.contains(EpollEvents::ONESHOT) { return Err(-EINVAL); }
     - if tgt_file.is_eventpoll() { return Err(-EINVAL); }
   }
7. if events.contains(EpollEvents::WAKEUP) && !current.has_cap(CAP_BLOCK_SUSPEND) {
     - ev.events &= !EpollEvents::WAKEUP.bits();
   }
8. /* cycle/nest check if target is epoll */
9. if tgt_file.is_eventpoll() {
     - if !Epoll::reverse_check_acceptable(&ctx, &tgt_file, EP_MAX_NESTS)? {
       - return Err(-ELOOP);
     }
   }
10. /* per-user watch quota */
11. if current.user_ns.epoll_watches() >= sysctl.fs_epoll_max_user_watches() {
      - return Err(-ENOSPC);
    }
12. /* allocate epitem */
13. let key = EpollKey { file: tgt_file.id(), fd };
14. let mut tree = ctx.rb_tree.write();
15. if tree.contains(&key) { return Err(-EEXIST); }
16. let epi = Arc::new(EpItem { rb_node, key, ep: ctx.clone(), event: RwLock::new(ev), ... });
17. tree.insert(key, epi.clone());
18. ctx.watches.fetch_add(1, AcqRel);
19. current.user_ns.epoll_watches_add(1);
20. /* register wait entry on target */
21. tgt_file.poll(&PollTable { proc: ep_ptable_queue_proc, item: epi.clone() });
22. /* if already-ready, link to rdllist */
23. if (epi.poll_state() & ev.events) != 0 { ctx.rdllist.lock().push_back(epi); ctx.wq.wake_one(); }
24. Ok(0).

`Epoll::modify(ctx, fd, tgt_file, ev_uptr)`:
1. if ev_uptr.is_null() { return Err(-EFAULT); }
2. let mut ev = copy_from_user::<EpollEvent>(ev_uptr)?;
3. let new_events = EpollEvents::from_bits(ev.events).ok_or(-EINVAL)?;
4. if new_events.contains(EpollEvents::EXCLUSIVE) { return Err(-EINVAL); }
5. let key = EpollKey { file: tgt_file.id(), fd };
6. let epi = ctx.rb_tree.read().get(&key).ok_or(-ENOENT)?.clone();
7. let prev = epi.event.write();
8. ev.events |= EpollEvents::ERR.bits() | EpollEvents::HUP.bits();
9. *prev = ev;
10. if (epi.poll_state() & ev.events) != 0 { ctx.rdllist.lock().push_back(epi.clone()); ctx.wq.wake_one(); }
11. Ok(0).

`Epoll::remove(ctx, fd, tgt_file)`:
1. let key = EpollKey { file: tgt_file.id(), fd };
2. let mut tree = ctx.rb_tree.write();
3. let epi = tree.remove(&key).ok_or(-ENOENT)?;
4. /* unregister wait entry */
5. epi.waiter.remove_from_target();
6. /* drop from ready list */
7. ctx.rdllist.lock().remove(&epi);
8. ctx.watches.fetch_sub(1, AcqRel);
9. current.user_ns.epoll_watches_sub(1);
10. Ok(0).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `op_known` | INVARIANT | unknown op ⟹ -EINVAL |
| `epfd_self` | INVARIANT | epfd == fd ⟹ -EINVAL |
| `add_unique` | INVARIANT | second ADD with same key ⟹ -EEXIST |
| `mod_existing` | INVARIANT | MOD on absent key ⟹ -ENOENT |
| `del_existing` | INVARIANT | DEL on absent key ⟹ -ENOENT |
| `exclusive_on_add_only` | INVARIANT | EXCLUSIVE on MOD/DEL ⟹ -EINVAL |
| `nest_bounded` | INVARIANT | epoll-on-epoll depth ≤ 4 |
| `watch_charge_balanced` | INVARIANT | ADD/DEL increment/decrement balanced |
| `wait_entry_registered` | INVARIANT | ADD ⟹ poll-wait entry on target |
| `wait_entry_unregistered` | INVARIANT | DEL ⟹ poll-wait entry removed |

### Layer 2: TLA+

`uapi/epoll_ctl.tla`:
- Variables per-instance: `interest_set : Map[key → mask]`, `ready_set`, `watches_counter`.
- Properties:
  - `safety_add_unique` — no duplicate keys.
  - `safety_charge_balanced` — `Σ adds == Σ dels + interest_set.size`.
  - `safety_wait_entry_match` — every key has exactly one wait-entry registered on target.
  - `safety_nest_bounded` — depth ≤ 4.
  - `liveness_terminate` — every ctl call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `add` post(ok): key present in tree, watches += 1 | `Epoll::add` |
| `modify` post(ok): event reflected; watches unchanged | `Epoll::modify` |
| `remove` post(ok): key absent; watches -= 1; wait-entry removed | `Epoll::remove` |
| `ctl` post(err): no partial state mutation | `Epoll::ctl` |

### Layer 4: Verus/Creusot functional

Per-`epoll_ctl(2)` man page, `man epoll(7)`, `Documentation/admin-guide/epoll.rst` semantic equivalence. Per-LTP `testcases/kernel/syscalls/epoll_ctl/*.c` and glibc `sysdeps/unix/sysv/linux/epoll_ctl.c` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

epoll_ctl reinforcement:

- **Per-user watch quota enforced on ADD** — defense against per-user watch exhaustion.
- **Per-instance mutex strict** — defense against per-mutate race producing dangling wait entries.
- **Per-cycle/nest check strict** — defense against per-recursive-poll kernel stack overflow.
- **`EPOLLEXCLUSIVE` immutable post-ADD** — defense against per-thundering-herd policy bypass.
- **`EPOLLWAKEUP` cap-gated** — defense against per-unprivileged-suspend-inhibit.
- **Wait-entry add/remove strictly paired** — defense against per-stale-callback UAF on target close.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on epoll_ctl entry** — randomize kernel stack; epoll_ctl manipulates the per-instance RB-tree which is a known SLAB-grooming target for kernel exploit chains (CVE-2022-3910-class).
- **PaX UDEREF on `copy_from_user` of `struct epoll_event`** — enforces unsigned-only userspace deref; refuses kernel pointers masqueraded as `event *`; the `epoll_data_t` union has a `void *ptr` member that is an attacker-controlled 64-bit opaque blob — UDEREF closes the historical "treat data.ptr as kernel pointer" type-confusion variants.
- **GRKERNSEC_BPF_HARDEN considerations** — the interest list manipulated by `epoll_ctl` is a user-controlled kernel-resident table acting as a small VM (each entry runs a poll callback). Apply BPF-style discipline: per-uid quota on interest list growth rate; audit every ADD/MOD with `current->comm`, `pid`, `fd`, `events`.
- **CAP_BPF gating for `EPOLLEXCLUSIVE` under hardened policy** — `EPOLLEXCLUSIVE` is a sharp scheduling primitive (overrides default wakeup fairness). Under hardened policy require `CAP_BPF` (or `CAP_SYS_NICE`); mirrors grsec's scheduler-affecting-flag doctrine.
- **GRKERNSEC_FIFO scaled to epoll-ctl flood** — per-uid rate-limit on `epoll_ctl` calls per second; under flood (rapid ADD/DEL churn to groom the SLAB) refuse with `-EAGAIN` and audit-log; defense against per-uid epitem-allocator-grooming.
- **PaX UDEREF for `data.ptr` propagation** — when `data.ptr` is reflected later through `epoll_wait`, ensure the reflection path treats it as an opaque user-controlled blob and never dereferences it kernel-side.
- **Strict nest-depth honoring** — `EP_MAX_NESTS = 4` is a hard upper bound on epoll-on-epoll depth; even with `CAP_SYS_ADMIN`, refuse deeper nests; defense against per-nested-wakeup kernel stack overflow (historical CVE-2020-14305-class).
- **GRKERNSEC_HIDESYM on `data` field reflection** — under `kernel.kptr_restrict ≥ 2` the kernel does not log `data.u64` payloads in syslog/audit; user-controlled but observable across debug surfaces is a leak vector.
- **`EPOLLWAKEUP` strict CAP_BLOCK_SUSPEND** — refuse `EPOLLWAKEUP` from cgroups not permitted to inhibit suspend; defense against per-cgroup energy-policy bypass.
- **Cross-namespace target file rejection** — under hardened policy, refuse `ADD` whose target file inode belongs to a different mount namespace (or with crosslink policy: different security context); defense against per-fd-confusion across security boundaries.
- **Audit log on per-user `ENOSPC`** — repeated `ENOSPC` from `max_user_watches` exhaustion is characteristic of memory-pressure-as-attack; rate-limited audit at `LOGLEVEL_WARNING`.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `epoll_create1.md` sibling — instance creation
- `epoll_pwait2.md` sibling — readiness retrieval
- `fs/eventpoll.md` Tier-3 — RB-tree, ready-list, callback path
- Implementation code
