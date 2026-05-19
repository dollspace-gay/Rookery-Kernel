---
title: "Tier-5 syscall: wait4(2) — syscall 61"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`wait4(2)` blocks the caller until one of its children changes state, then reports the child's status, optionally including a `rusage` accounting structure. It is the BSD-derived ancestor of `waitid(2)` and the superset of `wait(2)` and `waitpid(2)` (which are libc wrappers, not syscalls on most Linux ABIs). The four arguments expose: which child to wait for (`pid`), where to write the encoded status (`wstatus`), which transitions to report (`options`), and where to write resource-usage accounting (`rusage`).

Critical for: every shell, every init system, every container supervisor, every process-pool runtime. A correct wait4 loop is what prevents zombie accumulation; an incorrect one is the canonical PID-leak bug.

### Acceptance Criteria

- [ ] AC-1: `fork` + child `_exit(0)` + `wait4(-1, &s, 0, NULL)`: returns tgid, `WIFEXITED && WEXITSTATUS == 0`.
- [ ] AC-2: Child killed by SIGSEGV: `WIFSIGNALED && WTERMSIG == SIGSEGV`.
- [ ] AC-3: `wait4(-1, &s, WNOHANG, NULL)` no ready child: returns 0.
- [ ] AC-4: `wait4(123, ...)` with non-child 123: `-ECHILD`.
- [ ] AC-5: `wait4(-1, NULL, 0, NULL)` blocks until child reports.
- [ ] AC-6: `wait4(0, ...)` / `wait4(-pgid, ...)` match by pgrp.
- [ ] AC-7: SIGINT without SA_RESTART during sleep: `-EINTR`.
- [ ] AC-8: `WUNTRACED` returns on SIGSTOP'd child: `WIFSTOPPED && WSTOPSIG == SIGSTOP`.
- [ ] AC-9: `WCONTINUED` returns on SIGCONT'd child: `WIFCONTINUED`.
- [ ] AC-10: Unknown options bits: `-EINVAL`.
- [ ] AC-11: `rusage` populated with reaped child's CPU + RSS.
- [ ] AC-12: Two threads racing same child: one succeeds; other `-ECHILD`.
- [ ] AC-13: After reap, child TGID released; not in `/proc`.
- [ ] AC-14: `SIGCHLD = SIG_IGN` + child exits: auto-reaped; wait4 → `-ECHILD`.
- [ ] AC-15: Subreaper reaps orphaned grandchild after its parent dies.
- [ ] AC-16: AUDIT_CHILD_REAP emitted.
- [ ] AC-17: `WCOREDUMP` set when child dumped core.

### Architecture

```rust
#[syscall(nr = 61, abi = "sysv")]
pub fn sys_wait4(
    pid:     i32,
    wstatus: UserPtr<i32>,
    options: i32,
    rusage:  UserPtr<Rusage>,
) -> isize {
    Wait::do_wait4(pid, wstatus, options, rusage)
}
```

`Wait::do_wait4(pid, wstatus, options, rusage) -> isize`:
1. Validate `options & ~KNOWN == 0` (KNOWN = WNOHANG|WUNTRACED|WCONTINUED|__WCLONE|__WALL|__WNOTHREAD); else `-EINVAL`.
2. Build Selector from pid (Tgid/Pgrp(self)/Any/Pgrp(|-pid|)).
3. Loop:
4.   Some(child) = scan_children(self, sel, options) ⟹ consume_child; copy_out wstatus/rusage; audit::log_child_reap; return tgid.
5.   None ⟹ if !has_any_eligible_child ⟹ `-ECHILD`; if WNOHANG ⟹ return 0; sleep_interruptible on waitq.
6.   Wake::SignalDelivered ⟹ `-EINTR`; Wake::ChildChanged ⟹ continue.

`Wait::consume_child(child, options) -> (tgid, status, rusage)`:
1. Exiting(code, sig, dumped): encode_exit; collect_rusage; release_task (drop struct, free pid).
2. Stopped(stopsig) under WUNTRACED: encode_stop; mark Stopped-reported (report once).
3. Continued under WCONTINUED: transition to Running; status = 0xFFFF.
4. Other states unreachable — scan_children gates eligibility.

### Out of Scope

- `waitid(2)` (Tier-5 separate doc; richer semantics, used by glibc's `waitpid`).
- `wait(2)`/`waitpid(2)` libc wrappers (not syscalls on most ABIs).
- ptrace-wait4 interaction full state machine (Tier-3 in `kernel/ptrace.md`).
- `kernel/exit.c` release-task internals (Tier-3 in `kernel/exit.md`).
- Subreaper full topology (Tier-3 in `kernel/exit-reaper.md`).
- Implementation code.

### signature

```c
pid_t wait4(pid_t pid, int *wstatus, int options, struct rusage *rusage);
```

```c
struct rusage {
    struct timeval ru_utime;     /* user CPU time */
    struct timeval ru_stime;     /* system CPU time */
    long ru_maxrss;              /* max resident set (KiB) */
    long ru_ixrss;               /* (unmaintained) */
    long ru_idrss;
    long ru_isrss;
    long ru_minflt;              /* page reclaims (soft) */
    long ru_majflt;              /* page faults (hard) */
    long ru_nswap;
    long ru_inblock;
    long ru_oublock;
    long ru_msgsnd;
    long ru_msgrcv;
    long ru_nsignals;
    long ru_nvcsw;               /* voluntary context switches */
    long ru_nivcsw;              /* involuntary context switches */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pid` | `pid_t` | in | Selector. See "pid selector" below. |
| `wstatus` | `int *` | out | If non-NULL, encoded status; macros `WIFEXITED`, `WEXITSTATUS`, `WIFSIGNALED`, `WTERMSIG`, `WIFSTOPPED`, `WSTOPSIG`, `WIFCONTINUED` decode it. |
| `options` | `int` | in | Bitmask of `WNOHANG | WUNTRACED | WCONTINUED | __WCLONE | __WALL | __WNOTHREAD`. |
| `rusage` | `struct rusage *` | out | If non-NULL, receives the reaped child's resource accounting. |

### pid selector

```text
pid < -1      Wait for any child in process group |pid|.
pid == -1     Wait for any child.
pid == 0      Wait for any child in caller's process group.
pid > 0       Wait for the specific child with TGID == pid.
```

### return value

| Value | Meaning |
|---|---|
| `> 0` (TGID) | Child reaped (or its stop/continue reported). `*wstatus` set. |
| `0` | `WNOHANG` set and no eligible child has reported. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `ECHILD` | No eligible child (specified or any) exists in a waitable state, OR the specified `pid` is not a child of caller. |
| `EINTR`  | Wait was interrupted by an unblocked signal whose handler does not have `SA_RESTART`. |
| `EINVAL` | `options` contains unknown bits. |
| `EFAULT` | `wstatus` or `rusage` user pointer invalid. |

### abi surface

```text
__NR_wait4 (x86_64)  = 61
__NR_wait4 (i386)    = 114
__NR_wait4 (generic) = 260      /* arm64, riscv */

Options
  WNOHANG          0x00000001   /* return 0 if no child ready */
  WUNTRACED        0x00000002   /* report stopped children too */
  WCONTINUED      0x00000008   /* report SIGCONT-resumed children */
  WSTOPPED         WUNTRACED    /* POSIX alias */
  WEXITED          0x00000004   /* (waitid only; ignored by wait4) */
  WNOWAIT          0x01000000   /* (waitid only; ignored by wait4) */
  __WCLONE         0x80000000   /* match only clone-children (signal != SIGCHLD) */
  __WALL           0x40000000   /* match both clone and non-clone children */
  __WNOTHREAD      0x20000000   /* do not look at children of other threads in tgid */

Status encoding (32-bit)
  WIFEXITED(s)      (((s) & 0x7F) == 0)            /* low 7 bits zero ⟹ normal exit */
  WEXITSTATUS(s)    (((s) >> 8) & 0xFF)            /* exit code in bits 8..15 */
  WIFSIGNALED(s)    (((s) & 0x7F) > 0 && (((s) & 0x7F) < 0x7F))
  WTERMSIG(s)       ((s) & 0x7F)
  WCOREDUMP(s)      ((s) & 0x80)
  WIFSTOPPED(s)     (((s) & 0xFF) == 0x7F)         /* low 16 = (stopsig << 8) | 0x7F */
  WSTOPSIG(s)       WEXITSTATUS(s)                 /* in bits 8..15 */
  WIFCONTINUED(s)   ((s) == 0xFFFF)
  /* ptrace event = stopsig | (event << 16) */
```

### compatibility contract

REQ-1: Syscall number is **61** on x86_64; **260** on generic-syscall archs. ABI-stable.

REQ-2: Caller blocks until at least one eligible child reports a state change, unless `WNOHANG` is set.

REQ-3: pid selector: pid > 0 ⟹ child TGID == pid; pid == 0 ⟹ child in caller's pgrp; pid == -1 ⟹ any direct child; pid < -1 ⟹ child in pgrp |pid|. `-ECHILD` if no eligible.

REQ-4: Clone-signal selectors: `__WCLONE` matches children whose clone parent-death signal != SIGCHLD; `__WALL` matches both; default (neither) matches only SIGCHLD-children. `__WNOTHREAD` restricts to children of calling THREAD (not tgid-pooled).

REQ-5: State selectors: `WUNTRACED` reports group-stopped children (SIGSTOP/SIGTSTP/SIGTTIN/SIGTTOU); `WCONTINUED` reports SIGCONT-resumed children. Each transition reportable once.

REQ-6: `WNOHANG`: no block; return 0 if no eligible child reported.

REQ-7: Reap effects: success path drops the zombie's task struct, releases TID for reuse, detaches from parent's children list.

REQ-8: `SA_NOCLDWAIT` or `SIGCHLD = SIG_IGN`: children auto-reaped on exit; wait4 returns `-ECHILD`.

REQ-9: `rusage`: reports the reaped child's accounting only (not aggregated across siblings).

REQ-10: `wstatus` encoded per WIFEXITED/WIFSIGNALED/WIFSTOPPED/WIFCONTINUED; ptrace events in bits 16..23.

REQ-11: Signal-interrupt during sleep: handler without `SA_RESTART` ⟹ `-EINTR`; `*wstatus` unmodified.

REQ-12: Race-freedom: multiple threads racing for same child — exactly one succeeds; others see `-ECHILD` or continue blocking.

REQ-13: `pid` referring to a NON-child returns `-ECHILD` (not `-ESRCH`).

REQ-14: `PR_SET_CHILD_SUBREAPER`: subreaper becomes parent of orphaned descendants; wait4 on subreaper can reap them.

REQ-15: ptrace-event stops are reported to the *tracer's* wait4, not the parent's, unless `__WALL` and the tracer is also the parent.

REQ-16: AUDIT_CHILD_REAP emitted on success (reaped TGID, exit code, signal).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `options_validated` | INVARIANT | options & ~KNOWN == 0; else `-EINVAL`. |
| `pid_selector_total` | INVARIANT | every pid value classifies to exactly one Selector. |
| `child_reaped_exactly_once` | INVARIANT | concurrent wait4 race: at most one returns success per child. |
| `zombie_released_post_reap` | INVARIANT | post-success: child.state == Released. |
| `wnohang_nonblocking` | INVARIANT | WNOHANG ⟹ no schedule(); returns 0 or success or `-ECHILD`. |
| `signal_interrupts_wait` | INVARIANT | non-restart-handled signal ⟹ `-EINTR`. |
| `no_torn_status` | INVARIANT | wstatus and rusage both populated atomically with TGID return. |

### Layer 2: TLA+

`kernel/wait.tla`:
- States per child: {Running, Stopped(sig), Continued, Exiting(code, sig, dumped), Released}.
- States per parent: {Running, Waiting(selector, options)}.
- Properties:
  - `safety_one_reaper_per_child` — invariant.
  - `safety_zombie_freed_on_reap` — Exiting → Released atomically.
  - `safety_options_validated` — invariant.
  - `safety_nochld_after_all_reaped` — empty children list ⟹ `-ECHILD`.
  - `safety_signal_interrupts` — invariant.
  - `liveness_eventual_reap` — every Exiting child is eventually reaped (assuming parent calls wait4 fairly).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_wait4` post: returned tgid corresponds to a once-eligible child of caller | `Wait::do_wait4` |
| `consume_child` post: child transitions to Released (or reported-once for stop/cont) | `Wait::consume_child` |
| `scan_children` post: returns Some(c) iff c is eligible under selector+options | `Wait::scan_children` |
| `encode_exit` post: WIFEXITED/WIFSIGNALED/WCOREDUMP macros decode correctly | `Wait::encode_exit` |

### Layer 4: Verus / Creusot functional

Per-`wait4(2)` man-page semantic equivalence. LTP `wait4_01..wait4_06`, `waitpid01..waitpid13` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`wait4(2)` reinforcement:

- **Per-options validation** — defense against per-future-flag accidental matching.
- **Per-`__WNOTHREAD` scope** — defense against per-cross-thread reap confusion.
- **Per-single-reaper invariant** — defense against per-double-reap race causing UAF on task struct.
- **Per-`ECHILD` vs `EINTR` disambiguation** — defense against per-zombie-flood DoS retry storms (caller can distinguish "no children" from "signal interrupt").
- **Per-zombie released on reap** — defense against per-PID-table exhaustion.
- **Per-`SIGCHLD = SIG_IGN` auto-reap** — defense against per-zombie buildup in daemons that ignore SIGCHLD.
- **Per-subreaper inheritance** — defense against per-orphan-leak after parent death.
- **Per-AUDIT_CHILD_REAP** — defense against per-reap log elision.

### grsecurity / pax surface

- **PaX UDEREF on `wstatus`/`rusage` copy_to_user** — defense against per-buffer kernel-deref bug.
- **PAX_RANDKSTACK at wait4 entry** — randomizes kernel stack offset.
- **GRKERNSEC_HARDEN_PTRACE** — wait4 on a ptrace'd child is gated by grsec process-isolation policy; cross-uid waiters denied even with `CAP_SYS_PTRACE` if RBAC says so.
- **GRKERNSEC_PROC restrictions** — when grsec hides task X from caller's view in /proc, wait4 returning `-ECHILD` is consistent with the hide (no enumeration via wait4).
- **GRKERNSEC_BRUTE** — child killed by SIGSEGV/SIGBUS during repeated fork-bomb pattern: grsec counter incremented; sustained pattern triggers per-uid lockout.
- **Per-grsec wait4 audit** — every reap of a setuid-binary process logged with full status (catch privileged-process crashes for forensics).
- **PaX KERNEXEC on exit path** — `release_task` runs in KERNEXEC-mapped code; no R/W gadget for stale task-struct exploitation.
- **GRKERNSEC_CHROOT_FINDTASK** — chroot'd waiter cannot wait4 on tasks outside its chroot, even if they are descendants (escape-prevention).
- **Per-grsec subreaper RBAC** — `PR_SET_CHILD_SUBREAPER` itself may be restricted to RBAC-permitted subjects; wait4 honors the resulting topology.

