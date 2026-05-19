# Tier-5 UAPI: include/uapi/linux/wait.h — wait(2)/waitid(2)/waitpid(2) ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/wait.h (~23 lines)
  - kernel/exit.c (do_wait, kernel_waitid, wait_consider_task)
  - include/uapi/asm-generic/siginfo.h (siginfo_t for waitid)
  - bits/waitstatus.h (glibc-defined WIFEXITED / WEXITSTATUS / ... macros over int status)
-->

## Summary

The `wait(2)` family blocks the caller until a child changes state and reports the change. POSIX surface: `wait(int *status)`, `waitpid(pid_t pid, int *status, int options)`, `waitid(idtype_t, id_t, siginfo_t*, int options)`. Linux-only: `wait3`, `wait4`. `idtype` selects matching scope (`P_ALL` / `P_PID` / `P_PGID` / `P_PIDFD`). `options` controls which state changes are reportable (`WEXITED` / `WSTOPPED` / `WCONTINUED`), whether the call may return without a child (`WNOHANG`), and whether the child is reaped (`WNOWAIT` polls without reaping). Linux-specific `__WCLONE` / `__WALL` / `__WNOTHREAD` and historical `WUNTRACED` (alias of `WSTOPPED`) control wait-on-clone-children, wait-on-any-children, and same-thread-group restrictions. The `int status` returned via `waitpid`/`wait3`/`wait4` is encoded per `bits/waitstatus.h` and decoded with `WIFEXITED` / `WEXITSTATUS` / `WIFSIGNALED` / `WTERMSIG` / `WCOREDUMP` / `WIFSTOPPED` / `WSTOPSIG` / `WIFCONTINUED`. `waitid` returns a structured `siginfo_t` (no encoded int). Critical for: process-lifecycle plumbing, shell job control, init/PID-1, container runtimes, pidfd-based supervision.

This Tier-5 covers `include/uapi/linux/wait.h` (~23 lines).

## ABI surface

| Constant / Type | Value | Purpose |
|---|---|---|
| `WNOHANG` | `0x00000001` | return immediately if no child changed |
| `WUNTRACED` | `0x00000002` | also report stopped children (waitpid legacy spelling) |
| `WSTOPPED` | `WUNTRACED` (0x00000002) | also report stopped children (waitid spelling) |
| `WEXITED` | `0x00000004` | report exited children (waitid; implicit in waitpid) |
| `WCONTINUED` | `0x00000008` | report SIGCONT-continued children |
| `WNOWAIT` | `0x01000000` | leave child reapable (poll only) |
| `__WNOTHREAD` | `0x20000000` | don't wait on children of other threads in group |
| `__WALL` | `0x40000000` | wait on all children (clone + native) |
| `__WCLONE` | `0x80000000` | wait only on clone children (non-SIGCHLD) |
| `P_ALL` | `0` | waitid scope: any child |
| `P_PID` | `1` | waitid scope: specific PID |
| `P_PGID` | `2` | waitid scope: specific PGID |
| `P_PIDFD` | `3` | waitid scope: pidfd |

Status macros (from `bits/waitstatus.h`; ABI-stable encoding the kernel writes):
- `WEXITSTATUS(s)` = `(((s) & 0xff00) >> 8)` — exit code if WIFEXITED.
- `WTERMSIG(s)` = `((s) & 0x7f)` — terminating signal if WIFSIGNALED.
- `WSTOPSIG(s)` = `WEXITSTATUS(s)` — stop signal if WIFSTOPPED.
- `WIFEXITED(s)` = `(WTERMSIG(s) == 0)` — child exited normally.
- `WIFSIGNALED(s)` = `(((signed char)(((s) & 0x7f) + 1) >> 1) > 0)` — child killed by signal.
- `WIFSTOPPED(s)` = `(((s) & 0xff) == 0x7f)` — child stopped (ptrace / SIGSTOP).
- `WIFCONTINUED(s)` = `((s) == 0xffff)` — child SIGCONT'd.
- `WCOREDUMP(s)` = `((s) & 0x80)` — combined with WIFSIGNALED: dumped core.

`waitid` `siginfo_t` fields populated:
- `si_signo` = `SIGCHLD`.
- `si_code` = `CLD_EXITED` / `CLD_KILLED` / `CLD_DUMPED` / `CLD_TRAPPED` / `CLD_STOPPED` / `CLD_CONTINUED`.
- `si_pid` = child PID (in the caller's PID namespace).
- `si_uid` = child's real UID.
- `si_status` = exit code or signal number.
- `si_utime` / `si_stime` = child user/system CPU time.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `wait4(2)` | per-rusage-aware reap | `Wait::wait4` |
| `waitid(2)` | per-structured reap | `Wait::waitid` |
| `waitpid(2)` | per-POSIX reap | `Wait::waitpid` |
| `do_wait()` | per-blocking loop | `Wait::do_wait` |
| `wait_consider_task()` | per-task eligibility | `Wait::consider_task` |
| `eligible_child()` | per-options match | `Wait::eligible_child` |
| `__WCLONE` / `__WALL` / `__WNOTHREAD` | per-thread-group scope | `WaitOptions::CLONE/ALL/NOTHREAD` |
| `P_PIDFD` | per-pidfd select | `Wait::resolve_pidfd` |

## Compatibility contract

REQ-1: `waitid(idtype, id, infop, options)`:
- `idtype` ∈ { P_ALL, P_PID, P_PGID, P_PIDFD } else EINVAL.
- `options` MUST include at least one of { WEXITED, WSTOPPED, WCONTINUED } else EINVAL.
- `options` allowed bits: WEXITED | WSTOPPED | WCONTINUED | WNOHANG | WNOWAIT | __WALL | __WCLONE | __WNOTHREAD. Other bits → EINVAL.
- `infop` (if non-NULL): siginfo_t written on success; zeroed if no child matched & WNOHANG.
- P_PIDFD: `id` interpreted as pidfd; ESRCH if pidfd refers to a non-child unless task is in same process group.
- ECHILD if no eligible child exists.

REQ-2: `waitpid(pid, statusp, options)`:
- `pid > 0` → P_PID; `pid == 0` → P_PGID (caller's pgrp); `pid == -1` → P_ALL; `pid < -1` → P_PGID(|pid|).
- `options` allowed bits: WNOHANG | WUNTRACED | WCONTINUED | __WALL | __WCLONE | __WNOTHREAD.
- Implicit WEXITED (always reports exits in waitpid family).
- `statusp` (if non-NULL): int status encoded per macro table above.

REQ-3: `wait4(pid, statusp, options, rusage)`:
- Same as waitpid + `rusage` (if non-NULL) gets child resource usage.

REQ-4: `wait3(statusp, options, rusage)` == `wait4(-1, statusp, options, rusage)`.

REQ-5: Per-WNOHANG: if no child currently reportable, return 0 (waitid: clear infop->si_pid; waitpid: return 0).

REQ-6: Per-WNOWAIT: child remains reportable in zombie state; subsequent wait without WNOWAIT reaps.

REQ-7: Per-WSTOPPED / WUNTRACED: stopped children (ptrace-stop or SIGSTOP-job-control-stop) reportable.

REQ-8: Per-WCONTINUED: child that received SIGCONT after stop is reportable once.

REQ-9: Per-WEXITED: exited children reportable. Required for waitid; implicit for waitpid.

REQ-10: Per-`__WALL`: wait on all children regardless of clone-signal (clone()/SIGCHLD parent).

REQ-11: Per-`__WCLONE`: wait only on children whose termination signal is **not** SIGCHLD (legacy clone-thread); ignored if `__WALL` is set.

REQ-12: Per-`__WNOTHREAD`: do not consider children created by other threads of the caller's thread group.

REQ-13: Per-P_PIDFD: requires pidfd opened with `pidfd_open(2)` or `clone3(CLONE_PIDFD)`; allows wait-by-pidfd that races safely with PID reuse.

REQ-14: Per-status encoding (kernel writes the same int that glibc decodes):
- Normal exit: `(exit_code & 0xff) << 8` ⟹ WIFEXITED true, WEXITSTATUS = exit_code.
- Signal kill: `sig | (core ? 0x80 : 0)` ⟹ WIFSIGNALED true, WTERMSIG = sig.
- Stop: `(sig << 8) | 0x7f` ⟹ WIFSTOPPED true, WSTOPSIG = sig.
- Continue: `0xffff` ⟹ WIFCONTINUED true.

REQ-15: Per-ECHILD: no eligible child matched after considering options + scope (including __WALL/__WCLONE/__WNOTHREAD filters).

REQ-16: Per-EINTR: pending signal interrupts a blocked wait (no WNOHANG).

## Acceptance Criteria

- [ ] AC-1: `waitid(P_PID, child, &info, WEXITED)` after `exit(7)` populates si_code=CLD_EXITED, si_status=7.
- [ ] AC-2: `waitid` with options==0 → EINVAL.
- [ ] AC-3: `waitid(P_PIDFD, pidfd, ...)` ESRCH if pidfd is not a child.
- [ ] AC-4: `waitpid(-1, &st, WNOHANG)` returns 0 if no child reportable.
- [ ] AC-5: WIFEXITED(st) ∧ WEXITSTATUS(st)==N after child exit(N).
- [ ] AC-6: WIFSIGNALED(st) ∧ WTERMSIG(st)==SIGSEGV ∧ WCOREDUMP(st) after segfault+core.
- [ ] AC-7: WIFSTOPPED(st) ∧ WSTOPSIG(st)==SIGSTOP under WUNTRACED.
- [ ] AC-8: WIFCONTINUED(st) under WCONTINUED after SIGCONT.
- [ ] AC-9: `WNOWAIT`: subsequent wait without WNOWAIT reaps the same child.
- [ ] AC-10: ECHILD when no eligible child.
- [ ] AC-11: EINTR when blocked wait is interrupted by signal.
- [ ] AC-12: `__WCLONE` returns clone-thread children; ignores SIGCHLD-parent native fork children.
- [ ] AC-13: `__WALL` overrides `__WCLONE`.
- [ ] AC-14: `__WNOTHREAD` excludes children of sibling threads.
- [ ] AC-15: waitpid pid<-1 matches PGID == -pid.

## Architecture

Rookery surface in `kernel/wait/uapi.rs`:

```rust
#[repr(i32)]
pub enum IdType {
    All   = 0,
    Pid   = 1,
    Pgid  = 2,
    Pidfd = 3,
}

bitflags! {
    pub struct WaitOptions: i32 {
        const NOHANG     = 0x00000001;
        const UNTRACED   = 0x00000002;  // alias STOPPED
        const STOPPED    = 0x00000002;
        const EXITED     = 0x00000004;
        const CONTINUED  = 0x00000008;
        const NOWAIT     = 0x01000000;
        const __NOTHREAD = 0x20000000;
        const __ALL      = 0x40000000;
        const __CLONE    = 0x80000000_u32 as i32;
    }
}

pub const VALID_WAITID:  WaitOptions = WaitOptions::EXITED.union(WaitOptions::STOPPED)
                                          .union(WaitOptions::CONTINUED)
                                          .union(WaitOptions::NOHANG)
                                          .union(WaitOptions::NOWAIT)
                                          .union(WaitOptions::__NOTHREAD)
                                          .union(WaitOptions::__ALL)
                                          .union(WaitOptions::__CLONE);

pub const VALID_WAITPID: WaitOptions = WaitOptions::NOHANG.union(WaitOptions::UNTRACED)
                                          .union(WaitOptions::CONTINUED)
                                          .union(WaitOptions::__NOTHREAD)
                                          .union(WaitOptions::__ALL)
                                          .union(WaitOptions::__CLONE);
```

`Wait::waitid(idtype, id, infop, options)`:
1. /* Validate */
2. if idtype ∉ {0,1,2,3}: return -EINVAL.
3. if options & !VALID_WAITID: return -EINVAL.
4. if !(options & (EXITED | STOPPED | CONTINUED)): return -EINVAL.
5. /* Resolve scope */
6. scope = match idtype {
     P_ALL  => Scope::Any,
     P_PID  => Scope::Pid(id),
     P_PGID => if id == 0 { Scope::Pgid(current.pgrp) } else { Scope::Pgid(id) },
     P_PIDFD => Scope::Pidfd(Self::resolve_pidfd(id)?),
   }.
7. /* Block-and-collect */
8. retval = Self::do_wait(scope, options, &mut infop_out).
9. if infop != NULL: copy_to_user(infop, &infop_out).
10. return retval.

`Wait::waitpid(pid, statusp, options) -> Pid`:
1. if options & !VALID_WAITPID: return -EINVAL.
2. scope = match pid {
     -1 => Scope::Any,
     0  => Scope::Pgid(current.pgrp),
     p if p > 0 => Scope::Pid(p),
     p          => Scope::Pgid(-p),
   }.
3. options |= EXITED. /* implicit */
4. (rv, sinfo) = Self::do_wait(scope, options, ...).
5. if rv > 0 ∧ statusp: copy_to_user(statusp, &encode_status(&sinfo)).
6. return rv.

`Wait::do_wait(scope, options, info_out) -> i32`:
1. loop:
   - any_eligible = false.
   - rcu_read_lock.
   - for child in current.iter_children_by_options(options):
     - if !scope.matches(child): continue.
     - any_eligible = true.
     - if Self::consider_task(child, options, info_out): return child.pid.
   - rcu_read_unlock.
   - if !any_eligible: return -ECHILD.
   - if options & NOHANG: info_out.zero(); return 0.
   - schedule_or_signal()? else return -EINTR.

`Wait::consider_task(child, options, info_out) -> bool`:
1. if child.exit_state == EXIT_ZOMBIE ∧ (options & EXITED):
   - info_out.populate(CLD_EXITED/CLD_KILLED/CLD_DUMPED, child).
   - if !(options & NOWAIT): release_task(child).
   - return true.
2. if child.is_stopped ∧ (options & (STOPPED|UNTRACED)):
   - info_out.populate(CLD_STOPPED, child).
   - return true.
3. if child.is_continued ∧ (options & CONTINUED):
   - info_out.populate(CLD_CONTINUED, child).
   - return true.
4. return false.

`encode_status(sinfo) -> i32`:
- CLD_EXITED: `(sinfo.si_status & 0xff) << 8`.
- CLD_KILLED: `sinfo.si_status & 0x7f`.
- CLD_DUMPED: `(sinfo.si_status & 0x7f) | 0x80`.
- CLD_STOPPED / CLD_TRAPPED: `(sinfo.si_status << 8) | 0x7f`.
- CLD_CONTINUED: `0xffff`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `waitid_idtype_valid` | INVARIANT | idtype ∈ {P_ALL, P_PID, P_PGID, P_PIDFD} |
| `waitid_options_subset` | INVARIANT | options ⊆ VALID_WAITID |
| `waitid_requires_event_bit` | INVARIANT | EXITED | STOPPED | CONTINUED non-zero |
| `waitpid_options_subset` | INVARIANT | options ⊆ VALID_WAITPID |
| `nowait_does_not_reap` | INVARIANT | NOWAIT ⟹ child remains zombie post-call |
| `status_encoding_roundtrip` | INVARIANT | decode(encode(sinfo)) == sinfo for {EXITED,KILLED,DUMPED,STOPPED,CONTINUED} |
| `eligible_child_scope_match` | INVARIANT | scope check precedes options check |
| `pidfd_not_child_esrch` | INVARIANT | P_PIDFD to non-child ⟹ ESRCH |

### Layer 2: TLA+

`uapi/wait.tla`:
- States: child ∈ { RUNNING, STOPPED, CONTINUED, ZOMBIE, REAPED }.
- Properties:
  - `safety_NOWAIT_no_reap` — NOWAIT call preserves ZOMBIE state.
  - `safety_one_continue_per_cont` — each SIGCONT yields exactly one WIFCONTINUED report.
  - `safety_pidfd_no_pid_reuse_race` — P_PIDFD never confused by recycled PID.
  - `liveness_zombie_reaped` — ZOMBIE child eventually REAPED after non-NOWAIT wait.
  - `liveness_block_or_ECHILD` — call either blocks, reaps, or returns ECHILD; never spins.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `waitid` post: infop populated ∨ rv ∈ {-EINVAL,-ECHILD,-EINTR,0} | `Wait::waitid` |
| `waitpid` post: statusp encoded per encode_status | `Wait::waitpid` |
| `consider_task` post: NOWAIT ⟹ task unchanged | `Wait::consider_task` |
| `do_wait` post: returns child.pid ∨ -errno | `Wait::do_wait` |
| `encode_status` post: monotone inverse of glibc waitstatus.h macros | `Wait::encode_status` |

### Layer 4: Verus/Creusot functional

Per-POSIX.1-2024 § wait/waitpid/waitid semantic equivalence. Per-LTP `testcases/kernel/syscalls/wait*/` and `glibc` `posix/tst-wait*.c` conformance.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

## Grsecurity/PaX-style Reinforcement

- **GRKERNSEC_HARDEN_PTRACE on WNOWAIT info-leak** — `waitid(..., WNOWAIT)` discloses child `si_uid`, `si_status`, `si_utime`/`si_stime` without reaping; treat as ptrace-class peek. Refuse `WNOWAIT` across UID/PID-namespace boundaries unless caller has `CAP_SYS_PTRACE` in the *child's* userns; eliminates a covert side-channel for sandboxed peers to read each other's exit codes.
- **P_PIDFD lifecycle strict** — `Wait::resolve_pidfd` MUST verify the pidfd's referent is alive (or a freshly reaped zombie owned by caller) under RCU; refuse `P_PIDFD` against a recycled-PID pidfd (race-window for cross-namespace pidfd holders) with `-ESRCH`. Audit-log `P_PIDFD` calls whose pidfd was opened by a different uid.
- **`__WALL` / `__WCLONE` deprecated under grsec** — emit a `pr_warn_ratelimited` whenever a non-uid-0 task uses `__WCLONE` or `__WALL`; these flags pre-date NPTL and are routinely misused by exploit chains to scoop transient zombie threads. Optionally gate behind `CAP_SYS_PTRACE`.
- **`__WNOTHREAD` enforced for sandboxed reapers** — when caller has `PR_SET_NO_NEW_PRIVS=1`, automatically OR `__WNOTHREAD` into options, preventing cross-thread accidental reaps in a hardened reactor process.
- **PAX_RANDKSTACK on wait entry** — randomize kernel stack on every `do_wait` blocking entry; prevents stack-layout disclosure via long-blocked wait timing.
- **`si_pid` namespace-scoped** — `Wait::consider_task` ALWAYS translates child pid via `task_pid_nr_ns(child, current->nsproxy->pid_ns)`; refuses to leak the global PID into an unprivileged pidns (defense against PID-ns escape primitive).
- **WCOREDUMP suppressed under suidexec** — if the child executed a suid binary, mask the WCOREDUMP bit and `CLD_DUMPED` si_code to prevent core-dump-existence-disclosure (combined with `fs.suid_dumpable=0`).
- **Block-and-restart guarded** — bounded number of EINTR/restart cycles per syscall to prevent signal-flood-DoS extending a blocked wait indefinitely without ever returning.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `kernel/exit.c` `do_exit`/`release_task`/`do_wait` implementation (covered in `kernel/exit.md` Tier-3)
- `bits/waitstatus.h` glibc macro source (out-of-tree; documented above for completeness)
- `clone3(CLONE_PIDFD)` and `pidfd_open(2)` (covered in `uapi/headers/sched.md` and `uapi/headers/pidfd.md`)
- `siginfo_t` full layout (covered in `uapi/headers/signal.md`)
- Implementation code
