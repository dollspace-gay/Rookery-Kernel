# Tier-5 syscall: poll(2) — syscall 7

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/select.c (SYSCALL_DEFINE3(poll), do_sys_poll, do_poll, poll_select_set_timeout)
  - include/linux/poll.h (struct poll_table_struct, struct poll_wqueues, poll_initwait)
  - include/uapi/asm-generic/poll.h (POLLIN/POLLOUT/POLLERR/POLLHUP/POLLPRI/POLLRDNORM/POLLWRNORM/POLLRDBAND/POLLWRBAND/POLLMSG/POLLREMOVE/POLLRDHUP/POLLNVAL)
  - arch/x86/entry/syscalls/syscall_64.tbl (7  common  poll)
-->

## Summary

`poll(2)` waits for one of a set of file descriptors to become ready for I/O. The caller supplies an array `fds[]` of `struct pollfd` whose `fd` and `events` fields name the descriptor and the bitmask of conditions of interest; on return the kernel writes the actually-occurring conditions to each entry's `revents`. `timeout` is in milliseconds: `0` polls non-blockingly, `-1` blocks indefinitely.

Unlike `select(2)`, `poll(2)` does not use fixed-size bitmaps and therefore scales beyond `FD_SETSIZE` (1024). It pre-dates and is superseded for high-fd-count workloads by `epoll(7)`, but remains the portable POSIX-blessed multiplexer used by libev, glib, qemu, ssh, X11, and almost every event-loop. Critical for: server frontends, terminal multiplexers, watch-dog daemons, and any program that needs both descriptor-readiness AND a sub-second timeout.

## Signature

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
    int   fd;       /* file descriptor; <0 disables this entry */
    short events;   /* requested events (POLL*) */
    short revents;  /* returned events (POLL*) */
};
```

```c
#define POLLIN       0x0001   /* readable */
#define POLLPRI      0x0002   /* urgent / OOB / priority data */
#define POLLOUT      0x0004   /* writable */
#define POLLERR      0x0008   /* error condition (output-only) */
#define POLLHUP      0x0010   /* hang up (output-only) */
#define POLLNVAL     0x0020   /* invalid fd (output-only) */
#define POLLRDNORM   0x0040   /* normal-band readable */
#define POLLRDBAND   0x0080   /* priority-band readable */
#define POLLWRNORM   0x0100   /* normal-band writable */
#define POLLWRBAND   0x0200   /* priority-band writable */
#define POLLMSG      0x0400   /* (Linux extension; STREAMS) */
#define POLLRDHUP    0x2000   /* peer half-closed (Linux) */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fds` | `struct pollfd *` | in/out | Array of `nfds` entries; `events` set by caller, `revents` written by kernel. |
| `nfds` | `nfds_t` (unsigned long) | in | Number of entries; bounded by `RLIMIT_NOFILE`. |
| `timeout` | `int` | in | Milliseconds; `0` = non-block, `<0` = infinite. |

## Return value

| Value | Meaning |
|---|---|
| `>0` | Number of `pollfd` entries with non-zero `revents`. |
| `0` | Timeout expired with no ready fds. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `fds` user buffer faults during copy_from_user / copy_to_user. |
| `EINTR` | Signal delivered before any fd became ready and before timeout. |
| `EINVAL` | `nfds > RLIMIT_NOFILE` (RLIMIT_NOFILE.rlim_cur) or `nfds > sysctl_nr_open`. |
| `ENOMEM` | Cannot allocate per-fd `poll_table_entry` slabs (for large `nfds`). |

Note: there is no analogue of `EBADF`; an invalid fd sets `POLLNVAL` in that entry's `revents` instead of failing the whole call.

## ABI surface

```text
__NR_poll   (x86_64)  = 7
__NR_poll   (arm64)   = absent — arm64 / riscv expose only ppoll(2) (271).
__NR_poll   (riscv)   = absent
__NR_poll   (i386)    = 168

/* Timeout granularity: caller-millisecond converted to ktime_t. */
/* Internally normalized via poll_select_set_timeout to an absolute deadline. */
```

## Compatibility contract

REQ-1: Syscall number is **7** on x86_64. Absent on arm64 / riscv; glibc emulates by calling `ppoll(2)` with a converted timespec.

REQ-2: `nfds` upper-bound: `RLIMIT_NOFILE.rlim_cur`. The kernel itself further caps at `sysctl_nr_open` for the user namespace.

REQ-3: For `nfds == 0`: `fds` is unused; poll degenerates to a sleep for `timeout` ms (returns 0).

REQ-4: For each `fds[i]` with `fd < 0`: entry is skipped, `revents` set to 0.

REQ-5: For each `fds[i]` with positive `fd` not pointing to an open file: `revents = POLLNVAL`; entry counts toward return value.

REQ-6: `events` bitmask is validated to known POLL* bits only via `EPOLLEX*` mask; unknown bits are silently masked off but never error.

REQ-7: `POLLERR`, `POLLHUP`, `POLLNVAL` are output-only; setting them in `events` is meaningless and silently masked. The kernel may set them in `revents` even when not requested.

REQ-8: Timeout semantics: `< 0` → infinite (interruptible only by signal); `0` → non-blocking poll; `> 0` → millisecond deadline; values > INT_MAX clamped to MAX_SCHEDULE_TIMEOUT.

REQ-9: Signal interruption: if a non-restartable signal is delivered before any fd is ready, `-EINTR`; per-SA_RESTART is **not** honored for poll(2) (BSD compatibility).

REQ-10: Per-pollwait registration: for each pollable fd the kernel calls `f_op->poll(file, &poll_table)`; the file driver installs a wait-queue entry. On wakeup the entry transitions the task back to runnable.

REQ-11: Per-fd refcount: each fd is `fget_light()`-ed for the duration of the call; large nfds cost O(nfds) refcount ops.

REQ-12: Per-architecture timekeeping: `do_poll` uses monotonic time (`CLOCK_MONOTONIC`) for deadline arithmetic; settimeofday does not affect outstanding poll calls.

REQ-13: Per-POLL_BUSY_LOOP: if `events` contains `POLLIN | POLLOUT` and `SO_BUSY_POLL` set on socket, the kernel may busy-spin briefly before sleeping.

REQ-14: Per-restart on EINTR: poll(2) is **not restarted** by `SA_RESTART` (POSIX-mandated); glibc handles retry in user-space if requested.

## Acceptance Criteria

- [ ] AC-1: poll over a pipe with data: returns 1, fds[0].revents has POLLIN.
- [ ] AC-2: poll with timeout=0 over an empty pipe: returns 0.
- [ ] AC-3: poll with timeout=-1 over an empty pipe interrupted by SIGINT: returns -1, EINTR.
- [ ] AC-4: poll on a closed fd: revents has POLLNVAL; result counts entry.
- [ ] AC-5: poll on a half-closed socket (peer closed write): revents has POLLRDHUP.
- [ ] AC-6: nfds > RLIMIT_NOFILE returns -1, EINVAL.
- [ ] AC-7: poll(NULL, 0, 100) sleeps ~100 ms then returns 0.
- [ ] AC-8: Unknown events bits in fds[i].events do not error; just ignored.
- [ ] AC-9: poll on a regular file fd: revents = POLLIN | POLLOUT (regular files always ready).
- [ ] AC-10: A wakeup before timeout: returns >0; timeout not exhausted (verified by monotonic-clock delta < timeout).

## Architecture

```rust
#[syscall(nr = 7, abi = "sysv")]
pub fn sys_poll(fds: UserPtr<PollFd>, nfds: u64, timeout_ms: i32) -> isize {
    let to = PollSelect::ms_to_ktime(timeout_ms);
    Poll::do_sys_poll(fds, nfds, &to)
}
```

`Poll::do_sys_poll(uptr, nfds, deadline) -> isize`:
1. if nfds > current.rlim_nofile() { return Err(EINVAL); }
2. let mut walk = PollListAlloc::new(nfds)?;       // ENOMEM for >POLL_STACK_ALLOC
3. unsafe { uptr.copy_in_array(&mut walk.entries, nfds)?; }   // EFAULT
4. let mut table = PollWqueues::init();
5. let ret = loop {
6.     let count = Poll::do_poll_pass(&mut walk, &mut table);
7.     if count > 0 || deadline.is_zero() { break Ok(count as isize); }
8.     match table.wait_until(deadline) {
9.         WaitResult::Ready  => continue,
10.        WaitResult::Timed  => break Ok(0),
11.        WaitResult::Signal => break Err(EINTR),
12.    }
13. };
14. table.poll_freewait();
15. unsafe { uptr.copy_out_array(&walk.entries, nfds)?; }     // EFAULT
16. ret

`Poll::do_poll_pass(walk, table) -> u32`:
1. let mut ready = 0u32;
2. for entry in walk.entries.iter_mut() {
3.     if entry.fd < 0 { entry.revents = 0; continue; }
4.     let f = match Files::fget(entry.fd) {
5.         None => { entry.revents = POLLNVAL; ready += 1; continue; }
6.         Some(f) => f,
7.     };
8.     let mask = entry.events | POLLERR | POLLHUP | POLLNVAL;
9.     let r = f.f_op.poll(&f, table) & mask;
10.    if r != 0 { entry.revents = r; ready += 1; }
11.    Files::fput(f);
12. }
13. ready

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nfds_bounded_by_rlimit` | INVARIANT | nfds > RLIMIT_NOFILE ⟹ EINVAL early. |
| `userptr_copy_in_bounded` | INVARIANT | copy_in_array exactly nfds * sizeof(pollfd). |
| `wqueues_freewait_always` | INVARIANT | poll_freewait called on every exit path. |
| `negative_fd_skipped` | INVARIANT | fd < 0 ⟹ revents = 0, no fget. |
| `pollnval_on_closed_fd` | INVARIANT | fget == None ⟹ revents = POLLNVAL. |

### Layer 2: TLA+

`fs/poll-syscall.tla`:
- States: per-validate, per-copy-in, per-pass, per-wait, per-wake, per-copy-out.
- Properties:
  - `safety_freewait_no_leak` — wqueue entries always cleaned.
  - `safety_eintr_signal_only` — EINTR ⟹ signal pending.
  - `safety_no_use_after_fput` — file ref released after each pass.
  - `liveness_progress_or_timeout` — deadline expires ⟹ return 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_sys_poll` post: ret >= 0 ⟹ count matches revents != 0 | `Poll::do_sys_poll` |
| `do_poll_pass` post: ready == count(entries.revents != 0) | `Poll::do_poll_pass` |
| `ms_to_ktime` post: ms < 0 ⟹ KTIME_MAX | `PollSelect::ms_to_ktime` |

### Layer 4: Verus / Creusot functional

Per-poll(2) POSIX-2017 semantic equivalence; LTP `poll01..poll03` and glibc `tst-poll*` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`poll(2)` reinforcement:

- **Per-nfds RLIMIT_NOFILE strict** — defense against per-nfds-overflow allocation.
- **Per-stack-alloc cutoff (POLL_STACK_ALLOC=256B)** — defense against per-kstack-bloat.
- **Per-fput-on-every-exit** — defense against per-file-ref leak.
- **Per-wqueue-freewait-on-every-exit** — defense against per-wait-list leak / corrupted dlist.
- **Per-mask events to known POLL bits** — defense against per-bit-smuggling.
- **Per-EINTR no SA_RESTART** — defense against per-signal-loss (POSIX-correct).
- **Per-deadline monotonic clock** — defense against per-settime jitter.

## Grsecurity / PaX surface

- **PaX UDEREF on fds copy_from_user / copy_to_user** — defense against per-fds kernel-deref bug; SMAP forced; bounded by `nfds * sizeof(pollfd)`.
- **GRKERNSEC_RLIMIT_NOFILE** — grsec lowers default soft nofile to 1024 for non-CAP_SYS_RESOURCE; defense against per-poll-fdarray-amplification DoS.
- **PAX_USERCOPY_HARDEN** — pollfd array copy uses whitelisted slab (POLL_STACK_ALLOC fast-path is in kstack and excluded from usercopy).
- **GRKERNSEC_TPE poll on /dev or special files** — defense against per-TPE-bypass via poll-readiness side-channel for TPE-restricted files.
- **PAX_REFCOUNT on file refcount during fget_light** — defense against per-fput-double / per-overflow UAF.
- **Per-busy-poll capped (CAP_NET_ADMIN)** — defense against per-CPU-spin DoS; SO_BUSY_POLL ignored for non-cap callers.
- **POLLNVAL never silenced** — grsec audits POLLNVAL events at GRKERNSEC_AUDIT_GROUP=1; useful for detecting fd-confusion exploit attempts.
- **Per-signal pending probed via TIF_SIGPENDING only** — defense against per-spurious-EINTR injection.
- **GRKERNSEC_HIDESYM on f_op->poll vtable** — symbol not in kallsyms.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- ppoll(2) sigmask handling (covered in Tier-5 `ppoll.md`).
- epoll(7) facility (covered in Tier-5 epoll docs).
- Per-driver f_op->poll implementations (covered per-driver).
- Implementation code.
