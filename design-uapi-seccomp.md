---
title: "Tier-5 UAPI: include/uapi/linux/seccomp.h — seccomp BPF + notifier ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`seccomp` is the kernel's per-task syscall-filtering subsystem. Per `seccomp(2)` (or `prctl(PR_SET_SECCOMP, ...)`), userspace installs either a hard-coded strict filter (`SECCOMP_MODE_STRICT`) or a classic-BPF program (`SECCOMP_MODE_FILTER`) whose 32-bit return value selects per-syscall action: `KILL_PROCESS`, `KILL_THREAD`, `TRAP`, `ERRNO`, `USER_NOTIF`, `TRACE`, `LOG`, or `ALLOW`. Filters run over `struct seccomp_data` (nr, arch, instruction_pointer, args[6]). With `SECCOMP_FILTER_FLAG_NEW_LISTENER`, the kernel returns a *notification fd*; a supervisor reads `struct seccomp_notif` on that fd, then writes `struct seccomp_notif_resp` to allow / deny / inject-result / continue the syscall. Critical for: sandboxing (Chromium, Firefox, systemd, container runtimes, OpenSSH), unprivileged syscall mediation, and the entire userspace security-policy ABI.

This Tier-5 covers the full UAPI surface of `include/uapi/linux/seccomp.h` (~157 lines).

### Acceptance Criteria

- [ ] AC-1: `seccomp(SECCOMP_SET_MODE_STRICT, 0, NULL)` succeeds for a single-threaded task; subsequent `openat` ⇒ SIGKILL.
- [ ] AC-2: `seccomp(SECCOMP_SET_MODE_STRICT, 1, NULL)` ⇒ -EINVAL.
- [ ] AC-3: `seccomp(SECCOMP_SET_MODE_FILTER, ...)` without `PR_SET_NO_NEW_PRIVS` and without `CAP_SYS_ADMIN` ⇒ -EACCES.
- [ ] AC-4: `seccomp(SECCOMP_SET_MODE_FILTER, NEW_LISTENER, fprog)` returns positive fd; `read(fd)` ⇒ -EINVAL.
- [ ] AC-5: `seccomp(SECCOMP_GET_NOTIF_SIZES, 0, &sz)` writes sizes equal to kernel-build sizes of `seccomp_notif`, `_resp`, `seccomp_data`.
- [ ] AC-6: `seccomp(SECCOMP_GET_ACTION_AVAIL, 0, &act=ALLOW)` ⇒ 0.
- [ ] AC-7: `seccomp(SECCOMP_GET_ACTION_AVAIL, 0, &act=USER_NOTIF)` ⇒ 0 (kernel ≥ 5.0 build).
- [ ] AC-8: `seccomp_data.nr/arch/instruction_pointer/args[6]` populated for the trapped syscall.
- [ ] AC-9: `RET_KILL_PROCESS` kills entire thread-group via SIGKILL.
- [ ] AC-10: `RET_KILL_THREAD` kills only calling thread via SIGSYS (uncatchable).
- [ ] AC-11: `RET_TRAP` delivers catchable SIGSYS with `si_errno = data`.
- [ ] AC-12: `RET_ERRNO` causes syscall to return -(data & 0xffff), clamped to MAX_ERRNO.
- [ ] AC-13: `RET_USER_NOTIF` blocks syscall; supervisor receives notification with matching `nr`, `arch`, `args`.
- [ ] AC-14: `RET_TRACE` with no tracer attached behaves like `ALLOW`.
- [ ] AC-15: `RET_LOG` executes syscall + writes AUDIT_SECCOMP record.
- [ ] AC-16: `RET_ALLOW` executes syscall normally.
- [ ] AC-17: Composite of `ALLOW + KILL_PROCESS` = `KILL_PROCESS`.
- [ ] AC-18: Composite of `USER_NOTIF + USER_NOTIF` = notification delivered to most-recently-added listener.
- [ ] AC-19: `SECCOMP_IOCTL_NOTIF_RECV` blocks until queued notification or returns -EAGAIN with O_NONBLOCK.
- [ ] AC-20: `SECCOMP_IOCTL_NOTIF_SEND` with matching id and `val/error` completes the trapped syscall.
- [ ] AC-21: `SECCOMP_IOCTL_NOTIF_SEND` with `FLAG_CONTINUE` lets the trapped syscall actually execute.
- [ ] AC-22: `SECCOMP_IOCTL_NOTIF_SEND` with stale id ⇒ -ENOENT.
- [ ] AC-23: `SECCOMP_IOCTL_NOTIF_ID_VALID` returns 0 for in-flight id, -ENOENT for completed/stale.
- [ ] AC-24: `SECCOMP_IOCTL_NOTIF_ADDFD` installs `srcfd` into the supervised task and returns the remote fd number.
- [ ] AC-25: `_NOTIF_ADDFD(flags=SETFD, newfd=N)` places the fd at slot N atomically replacing any existing fd.
- [ ] AC-26: `_NOTIF_ADDFD(flags=SEND)` atomically installs fd + completes notification with new fd as return value.
- [ ] AC-27: `SECCOMP_IOCTL_NOTIF_SET_FLAGS(FD_SYNC_WAKE_UP)` enables synchronous supervisor wake.
- [ ] AC-28: `FILTER_FLAG_TSYNC` syncs filter to all threads of caller's tgroup or -ESRCH.
- [ ] AC-29: `FILTER_FLAG_TSYNC_ESRCH` returns tid of failing thread (positive) instead of -ESRCH.
- [ ] AC-30: `FILTER_FLAG_WAIT_KILLABLE_RECV` causes pending notifications to be interruptible only by fatal signals.
- [ ] AC-31: `FILTER_FLAG_SPEC_ALLOW` opts out of SSBD enforcement for this filter install.
- [ ] AC-32: BPF verifier rejects loads from `seccomp_data` offset ≥ sizeof(seccomp_data).
- [ ] AC-33: BPF verifier rejects programs without a `BPF_RET` on every path.
- [ ] AC-34: Closing listener fd while supervised task is blocked in `RET_USER_NOTIF`: trapped syscall returns -ENOSYS.
- [ ] AC-35: Filter inherited by `fork()` child; child trapped on same syscall sees same behavior.
- [ ] AC-36: `execve` preserves filters iff `PR_SET_NO_NEW_PRIVS` set or installed with CAP_SYS_ADMIN.
- [ ] AC-37: `SECCOMP_IOC_MAGIC` ≡ `'!'` ≡ 0x21; other magic ⇒ -ENOTTY.

### Architecture

```
enum SeccompMode { Disabled = 0, Strict = 1, Filter = 2 }

bitflags! FilterFlags : u64 {
  TSYNC                = 1 << 0,
  LOG                  = 1 << 1,
  SPEC_ALLOW           = 1 << 2,
  NEW_LISTENER         = 1 << 3,
  TSYNC_ESRCH          = 1 << 4,
  WAIT_KILLABLE_RECV   = 1 << 5,
}

enum Action {
  KillProcess = 0x80000000,
  KillThread  = 0x00000000,
  Trap        = 0x00030000,
  Errno       = 0x00050000,
  UserNotif   = 0x7fc00000,
  Trace       = 0x7ff00000,
  Log         = 0x7ffc0000,
  Allow       = 0x7fff0000,
}

struct SeccompData {
  nr: i32,
  arch: u32,                          // AUDIT_ARCH_*
  instruction_pointer: u64,
  args: [u64; 6],
}

struct SeccompNotif {
  id: u64,
  pid: u32,
  flags: u32,                         // reserved 0
  data: SeccompData,
}

struct SeccompNotifResp {
  id: u64,
  val: i64,
  error: i32,
  flags: u32,                         // SECCOMP_USER_NOTIF_FLAG_CONTINUE
}

struct SeccompNotifSizes {
  seccomp_notif: u16,
  seccomp_notif_resp: u16,
  seccomp_data: u16,
}

struct SeccompNotifAddfd {
  id: u64,
  flags: u32,                         // SECCOMP_ADDFD_FLAG_{SETFD, SEND}
  srcfd: u32,
  newfd: u32,
  newfd_flags: u32,
}
```

`Seccomp::syscall(op, flags, args) -> isize`:
1. Match `op`:
   - `SET_MODE_STRICT` → `Seccomp::set_mode_strict(flags, args)`
   - `SET_MODE_FILTER` → `Seccomp::set_mode_filter(flags, args)`
   - `GET_ACTION_AVAIL` → `Seccomp::get_action_avail(flags, args)`
   - `GET_NOTIF_SIZES` → `Seccomp::get_notif_sizes(flags, args)`
   - else: -EINVAL.
2. Per-op handler validates per-REQ-N.

`Seccomp::set_mode_strict(flags, args)`:
1. If `flags != 0` ∨ `args != NULL`: return -EINVAL.
2. If task.seccomp.mode == Strict: return 0 (idempotent).
3. If task.seccomp.mode == Filter: return -EINVAL.
4. Atomically set task.seccomp.mode = Strict, install hard-coded filter (allow {read, write, _exit, exit_group, sigreturn}, kill else).

`Seccomp::set_mode_filter(flags, args)`:
1. Validate `flags` ⊆ legal mask; else -EINVAL.
2. Pre-condition check: `task.no_new_privs || capable(CAP_SYS_ADMIN)`; else -EACCES.
3. `flags & NEW_LISTENER + TSYNC` and `task.thread_group.count > 1` → -EINVAL.
4. Copy `sock_fprog` from user; bound `len ≤ BPF_MAXINSNS`.
5. Copy instructions; run BPF verifier (no maps, no helpers, reads only `seccomp_data`).
6. Allocate `SeccompFilter`; prepend to `task.seccomp.filter_chain`.
7. If first filter: task.seccomp.mode = Filter.
8. If `flags & TSYNC`: synchronize across thread-group; on failure return -ESRCH (or tid with `TSYNC_ESRCH`).
9. If `flags & NEW_LISTENER`: allocate notification listener; return new fd.
10. If `flags & SPEC_ALLOW`: clear per-task SSBD enforcement.
11. If `flags & LOG`: mark filter for audit-eligibility.
12. If `flags & WAIT_KILLABLE_RECV`: set per-listener wait-killable flag.
13. Return 0 (or fd / tid).

`Seccomp::run_filter(syscall_nr, regs) -> Action`:
1. Build `SeccompData { nr, arch, ip, args }`.
2. action_full = max u32.
3. For each filter (newest first):
   - ret = bpf_run(filter, &data).
   - action_full = min(action_full, ret).      // signed min: KILL_PROCESS wins
4. Resolve action = action_full & 0x7fff0000 ∨ KILL_PROCESS bit.
5. Return Action plus data (lower 16 bits).

`Seccomp::dispatch_action(action, data, regs)`:
- `KillProcess`: send SIGKILL to thread-group, fill SIGSYS si_*, return.
- `KillThread`: send SIGSYS (uncatchable in seccomp ctx) to current.
- `Trap`: send SIGSYS catchable; si_errno = data.
- `Errno`: set regs.return_value = -(data & 0xffff) clamped to MAX_ERRNO; skip syscall.
- `UserNotif`: enqueue on most-recent filter's listener; block syscall.
- `Trace`: if tracer attached, ptrace_notify(PTRACE_EVENT_SECCOMP, data); else Allow.
- `Log`: audit_seccomp(nr, action, data); execute syscall.
- `Allow`: execute syscall.

`Seccomp::notif_ioctl(listener, cmd, arg)`:
- Validate `_IOC_TYPE(cmd) == SECCOMP_IOC_MAGIC ('!')`; else -ENOTTY.
- Switch on `_IOC_NR(cmd)`:
  - 0 RECV: dequeue notification → copy_to_user `seccomp_notif`.
  - 1 SEND: copy_from_user `seccomp_notif_resp`; complete notification.
  - 2 ID_VALID: check id in-flight.
  - 3 ADDFD: install supervisor's srcfd into supervised task.
  - 4 SET_FLAGS: update listener flags.

### Out of Scope

- BPF classic instruction set (`include/uapi/linux/filter.h` `struct sock_filter`, `struct sock_fprog`) — covered in `uapi/headers/filter.md`.
- `AUDIT_ARCH_*` enumeration (`include/uapi/linux/audit.h`) — covered in `uapi/headers/audit.md`.
- `prctl(PR_SET_NO_NEW_PRIVS, ...)`, `prctl(PR_SET_SECCOMP, ...)`, `prctl(PR_GET_SECCOMP, ...)` — covered in `uapi/headers/prctl.md`.
- `signalfd` / `SIGSYS` `siginfo_t` general layout (`include/uapi/asm-generic/siginfo.h`) — covered separately.
- `ptrace(PTRACE_O_TRACESECCOMP, ...)` ptrace integration — covered in `uapi/headers/ptrace.md`.
- Landlock ABI (`include/uapi/linux/landlock.h`) — separate Tier-5 doc.
- Implementation code (`kernel/seccomp.c`).

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `SECCOMP_MODE_DISABLED/STRICT/FILTER` | per-task mode | `seccomp::Mode` |
| `SECCOMP_SET_MODE_STRICT` | per-`seccomp(2)` op 0 | `seccomp::op::set_mode_strict` |
| `SECCOMP_SET_MODE_FILTER` | per-`seccomp(2)` op 1 | `seccomp::op::set_mode_filter` |
| `SECCOMP_GET_ACTION_AVAIL` | per-`seccomp(2)` op 2 | `seccomp::op::get_action_avail` |
| `SECCOMP_GET_NOTIF_SIZES` | per-`seccomp(2)` op 3 | `seccomp::op::get_notif_sizes` |
| `SECCOMP_FILTER_FLAG_*` | per-filter flags | `seccomp::FilterFlags` |
| `SECCOMP_RET_*` | per-action return code | `seccomp::Action` |
| `SECCOMP_RET_ACTION_FULL / _ACTION / _DATA` | per-action mask | `seccomp::action_mask` |
| `struct seccomp_data` | per-filter input | `SeccompData` |
| `struct seccomp_notif` | per-notif read | `SeccompNotif` |
| `struct seccomp_notif_resp` | per-notif response | `SeccompNotifResp` |
| `struct seccomp_notif_sizes` | per-`GET_NOTIF_SIZES` | `SeccompNotifSizes` |
| `struct seccomp_notif_addfd` | per-`IOCTL_NOTIF_ADDFD` | `SeccompNotifAddfd` |
| `SECCOMP_USER_NOTIF_FLAG_CONTINUE` | per-notif-resp flag | `seccomp::UserNotifFlags::CONTINUE` |
| `SECCOMP_USER_NOTIF_FD_SYNC_WAKE_UP` | per-notif-fd sync flag | `seccomp::FdSyncFlags` |
| `SECCOMP_ADDFD_FLAG_SETFD/SEND` | per-addfd flags | `seccomp::AddfdFlags` |
| `SECCOMP_IOCTL_NOTIF_RECV` | per-notif ioctl 0 | `seccomp::ioctl::notif_recv` |
| `SECCOMP_IOCTL_NOTIF_SEND` | per-notif ioctl 1 | `seccomp::ioctl::notif_send` |
| `SECCOMP_IOCTL_NOTIF_ID_VALID` | per-notif ioctl 2 | `seccomp::ioctl::notif_id_valid` |
| `SECCOMP_IOCTL_NOTIF_ADDFD` | per-notif ioctl 3 | `seccomp::ioctl::notif_addfd` |
| `SECCOMP_IOCTL_NOTIF_SET_FLAGS` | per-notif ioctl 4 | `seccomp::ioctl::notif_set_flags` |
| `SECCOMP_IOC_MAGIC` | ioctl class `'!'` | `seccomp::IOC_MAGIC` |

### abi surface (constants + structs)

### Mode (`seccomp.mode` / `prctl(PR_SET_SECCOMP, mode)`)

```
SECCOMP_MODE_DISABLED              = 0   /* seccomp is not in use. */
SECCOMP_MODE_STRICT                = 1   /* uses hard-coded filter. */
SECCOMP_MODE_FILTER                = 2   /* uses user-supplied filter. */
```

### Operations (`seccomp(2)` first argument)

```
SECCOMP_SET_MODE_STRICT            = 0
SECCOMP_SET_MODE_FILTER            = 1
SECCOMP_GET_ACTION_AVAIL           = 2
SECCOMP_GET_NOTIF_SIZES            = 3
```

### Flags for `SECCOMP_SET_MODE_FILTER` (`seccomp(2)` second argument)

```
SECCOMP_FILTER_FLAG_TSYNC               = 1UL << 0
SECCOMP_FILTER_FLAG_LOG                 = 1UL << 1
SECCOMP_FILTER_FLAG_SPEC_ALLOW          = 1UL << 2
SECCOMP_FILTER_FLAG_NEW_LISTENER        = 1UL << 3
SECCOMP_FILTER_FLAG_TSYNC_ESRCH         = 1UL << 4
SECCOMP_FILTER_FLAG_WAIT_KILLABLE_RECV  = 1UL << 5   /* Received notifications wait in killable state */
```

### Filter return-value layout

```
/*
 * All BPF programs return a 32-bit value.
 *  - Upper 16 bits = action (ordered least-to-most permissive, signed).
 *  - Lower 16 bits = action-specific data.
 * Composed filters: min_t() over per-filter return = least permissive action.
 */
SECCOMP_RET_KILL_PROCESS           = 0x80000000U   /* kill the process */
SECCOMP_RET_KILL_THREAD            = 0x00000000U   /* kill the thread */
SECCOMP_RET_KILL                   = SECCOMP_RET_KILL_THREAD   /* legacy alias */
SECCOMP_RET_TRAP                   = 0x00030000U   /* disallow and force SIGSYS */
SECCOMP_RET_ERRNO                  = 0x00050000U   /* returns an errno */
SECCOMP_RET_USER_NOTIF             = 0x7fc00000U   /* notifies userspace */
SECCOMP_RET_TRACE                  = 0x7ff00000U   /* pass to a tracer or disallow */
SECCOMP_RET_LOG                    = 0x7ffc0000U   /* allow after logging */
SECCOMP_RET_ALLOW                  = 0x7fff0000U   /* allow */

/* Masks for the return value sections. */
SECCOMP_RET_ACTION_FULL            = 0xffff0000U   /* includes KILL_PROCESS bit */
SECCOMP_RET_ACTION                 = 0x7fff0000U   /* legacy mask */
SECCOMP_RET_DATA                   = 0x0000ffffU   /* action-specific data */
```

### `struct seccomp_data`

```
/**
 * @nr: the system call number
 * @arch: AUDIT_ARCH_* value as defined in <linux/audit.h>
 * @instruction_pointer: at the time of the system call.
 * @args: up to 6 system call arguments always stored as 64-bit values
 *        regardless of the architecture.
 */
struct seccomp_data {
  int   nr;
  __u32 arch;
  __u64 instruction_pointer;
  __u64 args[6];
};
```

### `struct seccomp_notif_sizes`

```
struct seccomp_notif_sizes {
  __u16 seccomp_notif;        /* sizeof(struct seccomp_notif)        */
  __u16 seccomp_notif_resp;   /* sizeof(struct seccomp_notif_resp)   */
  __u16 seccomp_data;         /* sizeof(struct seccomp_data)         */
};
```

### `struct seccomp_notif`

```
struct seccomp_notif {
  __u64 id;                   /* opaque, kernel-assigned, per-notification */
  __u32 pid;                  /* tid of the notified task                 */
  __u32 flags;                /* unused; reserved                          */
  struct seccomp_data data;   /* the would-be syscall                      */
};
```

### `SECCOMP_USER_NOTIF_FLAG_CONTINUE`

```
SECCOMP_USER_NOTIF_FLAG_CONTINUE   = 1UL << 0
```
(The header documents this as TOCTOU-prone: the supervisor releases the syscall to actually run, and the supervised process may have rewritten user pointers in the interval. Must not be used as a security policy.)

### `struct seccomp_notif_resp`

```
struct seccomp_notif_resp {
  __u64 id;       /* must match seccomp_notif.id */
  __s64 val;      /* return value to fake (when not CONTINUE) */
  __s32 error;    /* negative errno to fake (0 means use val) */
  __u32 flags;    /* SECCOMP_USER_NOTIF_FLAG_CONTINUE */
};
```

### Notification-fd sync flag

```
SECCOMP_USER_NOTIF_FD_SYNC_WAKE_UP = 1UL << 0
```

### `struct seccomp_notif_addfd`

```
SECCOMP_ADDFD_FLAG_SETFD           = 1UL << 0  /* Specify remote fd       */
SECCOMP_ADDFD_FLAG_SEND            = 1UL << 1  /* Addfd and return atomically */

/**
 * @id: The ID of the seccomp notification
 * @flags: SECCOMP_ADDFD_FLAG_*
 * @srcfd: The local fd number
 * @newfd: Optional remote FD number if SETFD option is set, otherwise 0.
 * @newfd_flags: The O_* flags the remote FD should have applied
 */
struct seccomp_notif_addfd {
  __u64 id;
  __u32 flags;
  __u32 srcfd;
  __u32 newfd;
  __u32 newfd_flags;
};
```

### Notification-fd ioctls

```
SECCOMP_IOC_MAGIC                  = '!'           /* 0x21 */
SECCOMP_IO(nr)                     = _IO  (SECCOMP_IOC_MAGIC, nr)
SECCOMP_IOR(nr, type)              = _IOR (SECCOMP_IOC_MAGIC, nr, type)
SECCOMP_IOW(nr, type)              = _IOW (SECCOMP_IOC_MAGIC, nr, type)
SECCOMP_IOWR(nr, type)             = _IOWR(SECCOMP_IOC_MAGIC, nr, type)

SECCOMP_IOCTL_NOTIF_RECV           = _IOWR(SECCOMP_IOC_MAGIC, 0, struct seccomp_notif)
SECCOMP_IOCTL_NOTIF_SEND           = _IOWR(SECCOMP_IOC_MAGIC, 1, struct seccomp_notif_resp)
SECCOMP_IOCTL_NOTIF_ID_VALID       = _IOW (SECCOMP_IOC_MAGIC, 2, __u64)
SECCOMP_IOCTL_NOTIF_ADDFD          = _IOW (SECCOMP_IOC_MAGIC, 3, struct seccomp_notif_addfd)
SECCOMP_IOCTL_NOTIF_SET_FLAGS      = _IOW (SECCOMP_IOC_MAGIC, 4, __u64)
```

### compatibility contract

REQ-1 `seccomp(SECCOMP_SET_MODE_STRICT, flags=0, args=NULL)`:
- Argument: none beyond op.
- `flags` must be 0; non-zero ⇒ -EINVAL.
- Effect: current task → `SECCOMP_MODE_STRICT`; from this point only `read`, `write`, `_exit`, `sigreturn` (and `exit_group`) syscalls permitted, any other ⇒ SIGKILL.
- Idempotent: re-calling with strict on an already-strict task ⇒ 0.
- Cannot be downgraded; `prctl(PR_SET_SECCOMP, 0, ...)` to disable always fails.

REQ-2 `seccomp(SECCOMP_SET_MODE_FILTER, flags, fprog)`:
- Argument: `fprog` = `struct sock_fprog *` (BPF program: len + insns) from `<linux/filter.h>`.
- `flags` ⊆ `{TSYNC | LOG | SPEC_ALLOW | NEW_LISTENER | TSYNC_ESRCH | WAIT_KILLABLE_RECV}`.
- Pre-conditions:
  - Caller is in user-context.
  - Caller has `PR_SET_NO_NEW_PRIVS` set OR has `CAP_SYS_ADMIN`. Else -EACCES.
  - BPF program passes verifier (only loads from `seccomp_data`, no kernel pointers, bounded).
- Effect:
  - Filter appended to current task's filter-list (head ordering: newest first; composed via `min_t(action)`).
  - If first filter installed and mode == DISABLED, mode → FILTER.
- Returns:
  - 0 on success **unless** flags includes `NEW_LISTENER`.
  - With `NEW_LISTENER`: returns the new notification fd (positive int).
  - With `TSYNC` + multiple threads: returns -ESRCH on failure to sync.
  - With `TSYNC_ESRCH`: returns thread-id (positive) of the offending thread on -ESRCH cause, allowing detection.
- Flag constraints:
  - `TSYNC | NEW_LISTENER` mutually exclusive prior to kernel 5.7; current kernel permits combination iff caller is sole thread.
  - `WAIT_KILLABLE_RECV` requires `NEW_LISTENER`.

REQ-3 `seccomp(SECCOMP_GET_ACTION_AVAIL, flags=0, action_ptr)`:
- Argument: `__u32 *action_ptr` pointing to a candidate `SECCOMP_RET_*` value.
- Returns: 0 if kernel implements that action; -EOPNOTSUPP otherwise.
- Allows userspace to feature-detect `RET_USER_NOTIF`, `RET_LOG`, `RET_KILL_PROCESS`, etc.

REQ-4 `seccomp(SECCOMP_GET_NOTIF_SIZES, flags=0, sizes_ptr)`:
- Argument: `struct seccomp_notif_sizes *sizes_ptr` (out).
- Kernel writes `sizes->seccomp_notif`, `sizes->seccomp_notif_resp`, `sizes->seccomp_data` = current kernel build's ABI sizes.
- Allows userspace to allocate forward-compatible buffers.

REQ-5 `struct seccomp_data` semantics:
- `nr`: native syscall number (per `arch`).
- `arch`: `AUDIT_ARCH_*` constant identifying the syscall convention (X86_64, I386, AARCH64, ARM, PPC64LE, S390X, RISCV64, ...).
- `instruction_pointer`: PC at syscall instruction.
- `args[6]`: always 6 entries × 64 bits, regardless of natural ABI argument count. Unused tail entries zero-padded.

REQ-6 Filter return composition:
- Multiple filters per task evaluated newest-first.
- Composite action = `min_t(u32, action_full, ...)` over per-filter returns — i.e., least permissive **action** wins, treating `KILL_PROCESS` (0x80000000) as a signed negative ⇒ most-restrictive of all.
- `SECCOMP_RET_DATA` (lower 16 bits) of the *chosen* action is the one whose action won; ties broken by latest filter.

REQ-7 `SECCOMP_RET_KILL_PROCESS`:
- Kernel delivers `SIGKILL` to the **entire thread-group** (process).
- `SIGSYS` siginfo: `si_signo = SIGSYS`, `si_code = SYS_SECCOMP`, `si_syscall = nr`, `si_arch = arch`, `si_call_addr = ip`, `si_errno = data & SECCOMP_RET_DATA` (clamped).

REQ-8 `SECCOMP_RET_KILL_THREAD` (= `SECCOMP_RET_KILL`):
- Kernel delivers `SIGSYS` (uncatchable in seccomp context) to the **calling thread only**.

REQ-9 `SECCOMP_RET_TRAP`:
- Kernel delivers catchable `SIGSYS` to the calling thread (`si_code = SYS_SECCOMP`).
- `si_errno = data & SECCOMP_RET_DATA`.
- Process may install a `SIGSYS` handler and emulate the syscall.

REQ-10 `SECCOMP_RET_ERRNO`:
- Syscall short-circuits; return value = -(data & SECCOMP_RET_DATA), clamped to [0, MAX_ERRNO].
- No syscall body runs.

REQ-11 `SECCOMP_RET_USER_NOTIF`:
- Syscall blocks; new `struct seccomp_notif { id, pid, flags, data }` queued on the listener fd (must have been created via `NEW_LISTENER`).
- Per-syscall-instance unique `id` assigned monotonically.
- Supervisor responds via `SECCOMP_IOCTL_NOTIF_SEND` carrying `seccomp_notif_resp { id, val, error, flags }`.
- If supervisor disappears (listener fd closed): kernel returns -ENOSYS to the trapped syscall.

REQ-12 `SECCOMP_RET_TRACE`:
- If a `PTRACE_O_TRACESECCOMP`-attached tracer exists: notification stop with `PTRACE_EVENT_SECCOMP`, `PTRACE_GETEVENTMSG` returns `data & SECCOMP_RET_DATA`. Tracer may modify syscall args / skip.
- If no tracer attached: behaves like `SECCOMP_RET_ALLOW` for compatibility, **except** the data field is preserved.

REQ-13 `SECCOMP_RET_LOG`:
- Syscall executed; entry logged via audit subsystem (`AUDIT_SECCOMP` record).
- Requires `SECCOMP_FILTER_FLAG_LOG` and kernel `seccomp_actions_logged` sysctl to include the action.

REQ-14 `SECCOMP_RET_ALLOW`:
- Syscall executed normally; no log.

REQ-15 `SECCOMP_FILTER_FLAG_TSYNC`:
- After install, synchronize the filter onto all threads of the calling thread-group.
- If any thread cannot be synced (different filter state, ptraced, etc.): returns negative.
- With `TSYNC_ESRCH`: positive return value = tid of failing thread; without `_ESRCH`: returns -ESRCH (ambiguous).

REQ-16 `SECCOMP_FILTER_FLAG_LOG`:
- Mark this filter for `RET_LOG` logging eligibility.

REQ-17 `SECCOMP_FILTER_FLAG_SPEC_ALLOW`:
- Opt out of automatic speculation mitigation (`SSBD`) for this task.
- Recommended only for trusted workloads; default is to enable SSBD on first filter install.

REQ-18 `SECCOMP_FILTER_FLAG_NEW_LISTENER`:
- On success, return value is a fd referring to the new notification listener.
- The fd supports: `poll`/`epoll` (readable when a notification is queued), `read` (returns -EINVAL — must use ioctls), `ioctl(SECCOMP_IOCTL_NOTIF_*)`, `close`.
- One listener fd per filter.

REQ-19 `SECCOMP_FILTER_FLAG_TSYNC_ESRCH`:
- Modifies `TSYNC` semantics so that a positive return value reports the failing thread's tid.

REQ-20 `SECCOMP_FILTER_FLAG_WAIT_KILLABLE_RECV`:
- Notifications wait in killable state — only fatal signals interrupt the wait.
- Reduces spurious EINTR-loop in supervisor.

REQ-21 `SECCOMP_IOCTL_NOTIF_RECV`:
- Argument: `struct seccomp_notif *` (in/out; `id == 0` on call).
- Behavior:
  - If a notification is queued: kernel zeroes the supplied struct, fills `id, pid, flags, data`, returns 0.
  - If no notification queued: blocks (or, with `O_NONBLOCK` on listener fd, returns -EAGAIN).
  - If notified task dies before response: returns -ENOENT.
- After successful recv, the notification is "in flight" — supervisor must `_NOTIF_SEND` or `_NOTIF_ADDFD` with the same id.

REQ-22 `SECCOMP_IOCTL_NOTIF_SEND`:
- Argument: `struct seccomp_notif_resp *`.
- `id` must match a previously RECV'd notification; else -ENOENT.
- `flags` ⊆ `{SECCOMP_USER_NOTIF_FLAG_CONTINUE}`.
- If `flags & CONTINUE`: kernel releases the trapped syscall to actually execute; supervisor's `val/error` ignored.
- Else: syscall completes with `error` (negative errno) if non-zero, else `val` as return value.

REQ-23 `SECCOMP_IOCTL_NOTIF_ID_VALID`:
- Argument: `__u64 *id`.
- Returns 0 if the id refers to an in-flight notification (the task still exists and is still blocked); -ENOENT otherwise.
- Used by supervisor to recheck before reading `/proc/<pid>/...` to avoid TOCTOU.

REQ-24 `SECCOMP_IOCTL_NOTIF_ADDFD`:
- Argument: `struct seccomp_notif_addfd *`.
- Installs `srcfd` (a fd in supervisor) into the notified task's fd table.
- `flags & SECCOMP_ADDFD_FLAG_SETFD` ⇒ use `newfd` as exact target descriptor (replacing any existing fd at that slot). Else kernel allocates lowest free fd.
- `flags & SECCOMP_ADDFD_FLAG_SEND` ⇒ atomically install fd **and** complete the notification with the new fd number as the return value. Cannot be combined with subsequent `_NOTIF_SEND`.
- `newfd_flags` = `O_*` flags applied to the remote fd (e.g., `O_CLOEXEC`).
- Return: on success, the remote fd number; on error: negative errno.

REQ-25 `SECCOMP_IOCTL_NOTIF_SET_FLAGS`:
- Argument: `__u64 *flags`.
- Sets per-listener flags (currently only `SECCOMP_USER_NOTIF_FD_SYNC_WAKE_UP`).
- `SECCOMP_USER_NOTIF_FD_SYNC_WAKE_UP`: wake supervisor synchronously when a notification is queued (reduces latency at scheduling-cost).

REQ-26 Lifecycle invariants:
- Once a task enters `SECCOMP_MODE_STRICT` or `SECCOMP_MODE_FILTER`, it cannot change mode (no transitions back to DISABLED).
- Filters survive `execve` iff `PR_SET_NO_NEW_PRIVS` is set or filter installed with `CAP_SYS_ADMIN`.
- Filters are inherited by `fork`/`clone` children.
- Listener fd is **not** inherited by children of the supervised task; it lives in supervisor.
- Closing the listener fd: pending notifications complete with -ENOSYS in the supervised task.

REQ-27 Verifier constraints (BPF subset):
- Read-only memory: only `struct seccomp_data` accessible (offset ∈ [0, sizeof(seccomp_data))).
- No `BPF_LD | BPF_ABS` from packet data; no `BPF_LD | BPF_IND`.
- All program paths must reach `BPF_RET`.
- Program length ≤ `BPF_MAXINSNS` (4096).
- No kernel pointers, no helper calls, no maps.

REQ-28 ABI stability:
- Struct layouts (`seccomp_data`, `seccomp_notif`, `seccomp_notif_resp`, `seccomp_notif_sizes`, `seccomp_notif_addfd`) are frozen.
- Adding fields to `seccomp_notif` or `_resp` requires either a new ioctl number or the `seccomp_notif_sizes`-based forward-compat mechanism.
- `seccomp_data.args[6]` is fixed at 6 even on archs with fewer registers.

REQ-29 Composite action semantics for `RET_USER_NOTIF`:
- Per the header note: when multiple filters return `USER_NOTIF` on the same syscall, the **most recently added** filter takes precedence (its listener receives the notification).
- A subsequent filter responding with `CONTINUE` overrides earlier filters' `USER_NOTIF` / `TRACE` mediations — the kernel cannot enforce policy via `USER_NOTIF` in that case.

REQ-30 SIGSYS siginfo (returned for `RET_KILL_THREAD`, `RET_KILL_PROCESS` (group), `RET_TRAP`):
- `si_signo = SIGSYS`.
- `si_code = SYS_SECCOMP`.
- `si_call_addr = instruction_pointer`.
- `si_syscall = nr`.
- `si_arch = arch`.
- `si_errno`:
  - For `RET_TRAP`: lower 16 bits of return value (`SECCOMP_RET_DATA`).
  - For `RET_KILL_*`: 0.

REQ-31 Filter program reference counting:
- Filter struct is per-thread but reference-counted: forked threads share the chain until they install a new filter.
- Filters dropped only on last reference (last task in chain exits).

REQ-32 `SECCOMP_GET_ACTION_AVAIL` action ordering:
- The userspace-supplied `__u32` must equal a single legal `SECCOMP_RET_*` value (not an OR of multiples).
- Returning -EOPNOTSUPP must not leak kernel-build internal information beyond presence/absence.

REQ-33 `SECCOMP_IOC_MAGIC == '!'` (0x21):
- All notification-fd ioctls share this magic. Other magics ⇒ -ENOTTY.

REQ-34 `SECCOMP_FILTER_FLAG_NEW_LISTENER` + `SECCOMP_FILTER_FLAG_TSYNC` interaction:
- Older kernels: combination rejected.
- Current kernel: permitted only if caller is single-threaded; else -EINVAL.

REQ-35 No raw `read(2)` on listener fd: kernel returns -EINVAL — userspace **must** use ioctls.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mode_monotonic` | INVARIANT | per-task: Disabled → {Strict, Filter}; no reverse transitions. |
| `filter_flags_mask` | INVARIANT | per-set_mode_filter: flags ⊆ {TSYNC, LOG, SPEC_ALLOW, NEW_LISTENER, TSYNC_ESRCH, WAIT_KILLABLE_RECV}. |
| `nnp_required` | INVARIANT | per-set_mode_filter: caller has no_new_privs or CAP_SYS_ADMIN. |
| `bpf_only_seccomp_data` | INVARIANT | per-bpf-program: loads restricted to offset ∈ [0, sizeof(seccomp_data)). |
| `bpf_terminates` | INVARIANT | per-bpf-program: every path ends in BPF_RET; length ≤ 4096. |
| `action_composition_min` | INVARIANT | per-run_filter: composite = min over per-filter actions (signed, KILL_PROCESS wins). |
| `notif_id_unique` | INVARIANT | per-listener: notification ids strictly increasing per listener. |
| `notif_resp_id_match` | INVARIANT | per-NOTIF_SEND: response.id must match in-flight RECV'd id. |
| `addfd_send_atomic` | INVARIANT | per-NOTIF_ADDFD(SEND): install fd + complete notif in one critical section. |
| `ioc_magic_check` | INVARIANT | per-ioctl: _IOC_TYPE(cmd) == 0x21 else -ENOTTY. |
| `listener_close_releases` | INVARIANT | per-close(listener): pending notifications complete with -ENOSYS in supervised task. |
| `sigsys_si_fields_complete` | INVARIANT | per-RET_TRAP / RET_KILL: si_signo=SIGSYS, si_code=SYS_SECCOMP, si_syscall/si_arch/si_call_addr populated. |

### Layer 2: TLA+

`uapi/seccomp.tla`:
- Models per-task seccomp mode, filter chain, listener fd lifetime, notification id sequence.
- Properties:
  - `safety_mode_monotonic` — no DISABLED ← FILTER transition.
  - `safety_filter_inherited_on_fork` — child task's filter_chain ⊇ parent's at fork instant.
  - `safety_filter_persists_on_execve_iff_nnp_or_admin` — preservation condition holds.
  - `safety_strict_only_allows_4_syscalls` — Strict mode permits only {read, write, _exit, sigreturn} (and exit_group); all others → SIGKILL.
  - `safety_notif_id_one_to_one` — each in-flight notification id corresponds to at most one blocked syscall instance.
  - `safety_supervisor_response_completes_exactly_once` — NOTIF_SEND for an id transitions notification from in-flight → completed, never twice.
  - `safety_action_least_permissive_wins` — composite action = minimum across filter chain.
  - `liveness_recv_eventually_progresses` — every queued notification is eventually RECV'd or supervisor disappears.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Seccomp::set_mode_strict` post: mode == Strict ∧ filter_chain == strict-template | `Seccomp::set_mode_strict` |
| `Seccomp::set_mode_filter` post: mode == Filter ∧ |chain| ≥ 1 ∧ (if NEW_LISTENER: ret > 0 ∧ listener.attached) | `Seccomp::set_mode_filter` |
| `Seccomp::run_filter` post: returned Action ∈ {KillProcess, KillThread, Trap, Errno, UserNotif, Trace, Log, Allow} | `Seccomp::run_filter` |
| `Seccomp::dispatch_action(Allow)` post: syscall actually executed | `Seccomp::dispatch_action` |
| `Seccomp::dispatch_action(Errno)` post: regs.ret = -data clamped to MAX_ERRNO; syscall body not entered | `Seccomp::dispatch_action` |
| `Seccomp::notif_recv` post: notif.id == in-flight id; notif.data == seccomp_data of trapped syscall | `Seccomp::notif_recv` |
| `Seccomp::notif_send(CONTINUE)` post: trapped syscall released to execute; supervisor's val/error ignored | `Seccomp::notif_send` |
| `Seccomp::notif_addfd(SEND)` post: fd installed ∧ notification completed atomically | `Seccomp::notif_addfd` |
| `Seccomp::get_notif_sizes` post: sizes == compile-time sizeof of the three structs | `Seccomp::get_notif_sizes` |
| `Seccomp::get_action_avail` post: returns 0 iff kernel implements that action | `Seccomp::get_action_avail` |

### Layer 4: Verus/Creusot functional

Per `Documentation/userspace-api/seccomp_filter.rst`:
- `seccomp(2)` op-table semantic equivalence with upstream `kernel/seccomp.c`.
- Filter return-value composition equivalence with upstream `seccomp_run_filters` (signed min).
- Notifier ioctl semantics equivalence with upstream `seccomp_notify_*` functions.
- Listener-fd lifetime equivalence: supervisor close → trapped syscalls return -ENOSYS.
- SIGSYS siginfo equivalence with upstream `seccomp_send_sigsys`.
- BPF verifier accepts the same programs (per `seccomp_check_filter` constraints).

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

Seccomp-UAPI reinforcement:

- **Per-no_new_privs OR CAP_SYS_ADMIN gate** — defense against per-setuid-shell escape via filter-imposed ENOSYS forging.
- **Per-mode-monotonic** — defense against per-runtime-policy-relax.
- **Per-filter chain immutable head-only-prepend** — defense against per-mid-chain-tamper.
- **Per-BPF verifier strict allow-list (seccomp_data only)** — defense against per-arbitrary-memory-read.
- **Per-BPF program length bound (BPF_MAXINSNS = 4096)** — defense against per-quadratic-CPU exhaustion.
- **Per-action-composition signed-min** — defense against per-most-permissive accidental wins.
- **Per-listener fd one-per-filter** — defense against per-aliasing.
- **Per-listener close completes pending with -ENOSYS** — defense against per-leaked-supervisor leaving syscall blocked.
- **Per-NOTIF_SEND id-match strict** — defense against per-id-confusion bypass.
- **Per-NOTIF_ADDFD(SETFD) DAC checks** — defense against per-fd-injection privilege gain.
- **Per-NOTIF_ADDFD(SEND) atomic** — defense against per-TOCTOU between addfd and resp.
- **Per-USER_NOTIF_FLAG_CONTINUE documented-as-not-policy** — defense against per-policy-design-error.
- **Per-NEW_LISTENER + TSYNC mutual-exclusion when threaded** — defense against per-supervisor-race-on-clone.
- **Per-SIGSYS siginfo si_call_addr / si_syscall / si_arch populated** — enables forensic / policy-debugging.

### grsecurity/pax-style reinforcement

- **PaX UDEREF/USERCOPY** — all `copy_from_user` paths for `sock_fprog`, `seccomp_notif`, `seccomp_notif_resp`, `seccomp_notif_addfd`, `seccomp_notif_sizes` use hardened-usercopy with sized bounds; SMAP/SMEP must be active; BPF program payload validated against `BPF_MAXINSNS` before any kernel-side allocation.
- **GRKERNSEC_HIDESYM** — `AUDIT_SECCOMP` log records sanitize `instruction_pointer` against host kernel-symbol leak (only userland IP retained); `SIGSYS` si_call_addr is the userland faulting IP, never a kernel pointer.
- **GRKERNSEC_PERF_HARDEN** — per-filter / per-listener performance counters are not exposed via `perf_event_open`; `seccomp_actions_logged` sysctl rate-limited; supervisor cannot use seccomp listener as a side-channel against host scheduler.
- **PAX_RANDKSTACK** — kernel-stack base re-randomized at each syscall entry path that performs filter evaluation, so repeated `RET_TRAP` / `RET_USER_NOTIF` cannot leak stack-layout via timing or via tracer side-channels.
- **PAX_KERNEXEC** — compiled BPF programs (whether JITted or interpreted) live in NX or W^X memory: JIT pages are emitted as RX after write-then-seal; interpreted programs are stored as data-only and dispatched via in-kernel interpreter without indirect-branch-to-user-controlled-address; the filter-chain head-pointer is in writable kernel data but never executable.
- **GRKERNSEC_HARDEN_PTRACE** — `RET_TRACE` is only useful with `PTRACE_O_TRACESECCOMP`, and the tracer must satisfy `gr_ptrace_allowed(target)`; cross-uid / cross-gid tracer attach forbidden by default, so `RET_TRACE` cannot be weaponized to elevate.
- **CAP_SYS_ADMIN gate** — installing a filter requires `PR_SET_NO_NEW_PRIVS` OR `CAP_SYS_ADMIN`; `SECCOMP_FILTER_FLAG_SPEC_ALLOW` further gated on `CAP_SYS_ADMIN` (only privileged code may opt out of SSBD); `SECCOMP_IOCTL_NOTIF_ADDFD` requires the supervisor's UID dominate the supervised task's UID under grsec policy.
- **GRKERNSEC_KMEM** — seccomp does not expose `/dev/mem` / `/dev/kmem` but the principle applies: BPF program instructions are never reflected back to userspace (no read-back via `prctl(PR_GET_SECCOMP_FILTER)`), and `seccomp_data.instruction_pointer` cannot dereference host kernel memory because the BPF verifier rejects any load outside `seccomp_data` bounds.
- **Seccomp landlock-style restrictions** — seccomp itself is the primitive; recommended profile for a supervisor process: install a seccomp filter that allows only `ioctl(notif_fd, SECCOMP_IOCTL_NOTIF_*)`, `read`/`write`/`epoll_wait` on the listener, and `pidfd_open` on the supervised pid; combined with Landlock, the supervisor's filesystem access is restricted to a sandbox directory so a compromised supervisor cannot escalate via the supervised task's resources.
- **Per-`NEW_LISTENER` fd O_CLOEXEC default** — defense against per-exec-leak of supervisor power.
- **Per-listener-fd not-mmappable** — defense against per-mmap side-channel.
- **Per-listener-fd not-spliceable** — defense against per-splice loop attack.
- **Per-`NOTIF_ADDFD.newfd_flags` validated** — only `O_CLOEXEC` / `O_NONBLOCK` accepted; other bits ⇒ -EINVAL.
- **Per-`NOTIF_ADDFD.srcfd` revalidated** — supervisor's fd checked at addfd time, not at later send.
- **Per-`SECCOMP_RET_USER_NOTIF` TOCTOU warning** — kernel docs prohibit using as policy; Rookery emits klog warning if `SECCOMP_USER_NOTIF_FLAG_CONTINUE` is used by a non-root supervisor mediating a root-owned target.
- **Per-`RET_KILL_THREAD` SIGSYS is uncatchable in seccomp context** — defense against per-handler-evasion via signal mask.
- **Per-`RET_KILL_PROCESS` reaches every thread** — defense against per-thread-survives-policy-kill.
- **Per-`SECCOMP_GET_ACTION_AVAIL` returns -EOPNOTSUPP without revealing kernel build** — defense against per-fingerprint.
- **Per-`SECCOMP_GET_NOTIF_SIZES` static-from-kbuild** — defense against per-size-confusion bridging old userspace and new kernel.
- **Per-filter-chain bounded** — total chain length per task bounded; defense against per-OOM via unbounded filter install loop.
- **Per-`TSYNC` only when caller is sole-thread or all-threads-share-nnp** — defense against per-cross-thread privilege smuggle.
- **Per-`WAIT_KILLABLE_RECV` mandatory under grsec policy for supervisors of long-running guests** — reduces D-state lockup risk.
- **Per-`SPEC_ALLOW` audited** — every install with `SPEC_ALLOW` logged with caller's tgid/exe-path.

