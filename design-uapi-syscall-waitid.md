---
title: "Tier-5 syscall: waitid(2) — syscall 247"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`waitid(2)` is the modern, fine-grained child-status-reaping syscall: it generalizes `wait4(2)` by accepting an explicit `idtype` selector (`P_ALL`, `P_PID`, `P_PGID`, `P_PIDFD`), distinguishing exit/stop/continue events via individually-maskable `options` flags, and writing a full `siginfo_t` into userspace describing the reaped event. Unlike `wait4`, it can observe a stopped/continued child WITHOUT consuming the event (`WNOWAIT`), enabling cooperative debuggers and supervisors. The `P_PIDFD` case (added 5.4) lets a process wait on a `pidfd_open(2)` handle, avoiding all PID-reuse races. An optional `struct rusage` returns resource consumption at the same time, matching `wait4(2)`.

Critical for: glibc `waitpid`/`waitid` wrappers, supervisor/process-manager event loops (systemd, runit, s6), debugger event reaping (`ptrace`-using GDB), container runtimes (`runc`/`crun`/`youki`), POSIX shell job control, `posix_spawn` plumbing, language runtime supervisors (Go, Erlang, Rust `std::process`), pidfd-based zero-PID-reuse process supervisors.

### Acceptance Criteria

- [ ] AC-1: `P_PID` with a direct child's TGID returns 0 and fills `infop` with `si_code == CLD_EXITED` after the child exited normally.
- [ ] AC-2: `P_ALL | WEXITED | WNOHANG` returns 0 with `si_pid == 0` when no zombie exists.
- [ ] AC-3: `WNOWAIT` leaves the zombie reachable: a second `waitid` without `WNOWAIT` reaps it.
- [ ] AC-4: `WSTOPPED` observes a SIGSTOP'd child with `si_code == CLD_STOPPED`.
- [ ] AC-5: `WCONTINUED` observes a SIGCONT'd child with `si_code == CLD_CONTINUED`.
- [ ] AC-6: `P_PIDFD` with a valid pidfd reaps the corresponding TGID; closing the pidfd before reap still allows reap if caller is parent.
- [ ] AC-7: `P_PIDFD` with a thread (non-leader) pidfd returns `EINVAL`.
- [ ] AC-8: `options == 0` returns `EINVAL`.
- [ ] AC-9: `options` containing `__WCLONE` returns `EINVAL` (waitid rejects legacy bits).
- [ ] AC-10: `infop = bad-user-ptr` returns `EFAULT` and reaps NOTHING.
- [ ] AC-11: `idtype = 99` returns `EINVAL`.
- [ ] AC-12: PID-namespace inner reaper sees `si_pid` in inner-ns; outer reaper sees outer-ns PID.
- [ ] AC-13: Reaping the same zombie twice returns `ECHILD` the second time.
- [ ] AC-14: `EINTR` is returned when a non-SA_RESTART signal arrives during a blocking `waitid`.
- [ ] AC-15: `rusage` filled iff non-NULL; semantically matches `wait4(2)` for the same child.
- [ ] AC-16: `siginfo_t` is fully zeroed before population (no stale-stack leak).

### Architecture

```rust
#[syscall(nr = 247, abi = "sysv")]
pub fn sys_waitid(
    idtype: u32,
    id: u32,
    infop: UserPtr<SigInfo>,
    options: i32,
    rusage: UserPtr<RUsage>,
) -> KResult<i32> {
    Waitid::do_waitid(idtype, id, infop, options, rusage)
}
```

`Waitid::do_waitid(idtype, id, infop, options, rusage) -> KResult<i32>`:
1. /* Validate options: at least one of WEXITED/WSTOPPED/WCONTINUED; no legacy bits */
2. if options & !(WNOHANG|WUNTRACED|WEXITED|WCONTINUED|WNOWAIT) != 0: return Err(EINVAL).
3. if options & (WEXITED|WSTOPPED|WCONTINUED) == 0: return Err(EINVAL).
4. /* Pre-validate userspace pointers via PaX UDEREF */
5. if !infop.is_null(): access_ok_uderef(infop, sizeof::<SigInfo>())?
6. if !rusage.is_null(): access_ok_uderef(rusage, sizeof::<RUsage>())?
7. /* Resolve idtype/id into wait_opts.wo_pid */
8. let wo = WaitOpts::new(idtype, id, options)?
9. match idtype:
   - P_ALL: wo.pid = None.
   - P_PID: wo.pid = Some(find_get_pid(id)?).
   - P_PGID: if id == 0 { id = task_pgrp_vnr(current()); } wo.pid = Some(find_get_pid(id)?).
   - P_PIDFD: wo.pidfd = Some(fdget_pidfd(id)?); wo.pid = Some(pidfd_to_pid(wo.pidfd)?).
   - _: return Err(EINVAL).
10. /* Core wait loop */
11. let mut info: SigInfo = SigInfo::zeroed();
12. let mut ru: RUsage = RUsage::default();
13. let ret = Waitid::do_wait(&mut wo, &mut info, &mut ru)?;
14. /* Copy out atomically */
15. if !infop.is_null(): copy_to_user(infop, &info)?
16. if !rusage.is_null(): copy_to_user(rusage, &ru)?
17. Ok(ret)

`Waitid::do_wait(wo, info, ru) -> KResult<i32>`:
1. /* Iterate children/descendants matching wo */
2. loop:
   - for child in Waitid::eligible_children(current(), wo):
     - if let Some(event) = Waitid::consider(child, wo, info, ru)? { return Ok(0); }
   - /* No event */
   - if wo.options & WNOHANG: { info.zero(); return Ok(0); }
   - if !current().has_children_matching(wo): return Err(ECHILD).
   - schedule_until_sigchld_or_signal()?;   // may return ERESTARTSYS

`Waitid::consider(child, wo, info, ru) -> KResult<Option<()>>`:
1. /* LSM check */
2. security_task_wait(child)?;
3. match child.exit_state:
   - EXIT_ZOMBIE if wo.options & WEXITED:
     - info.populate_exit(child); ru.copy_from(child);
     - if !(wo.options & WNOWAIT): release_task(child);
     - Ok(Some(()))
   - STOPPED if wo.options & WSTOPPED:
     - info.populate_stopped(child); Ok(Some(()))
   - CONTINUED if wo.options & WCONTINUED:
     - info.populate_continued(child); if !(wo.options & WNOWAIT) { child.clear_continued(); } Ok(Some(()))
   - _: Ok(None)

`SigInfo::populate_exit(child)`:
1. self.si_signo = SIGCHLD.
2. self.si_errno = 0.
3. self.si_pid = pid_vnr(child); self.si_uid = child.cred.uid.
4. match child.exit_code:
   - normal-exit: self.si_code = CLD_EXITED; self.si_status = (code >> 8) & 0xff.
   - signal-killed: self.si_code = CLD_KILLED|CLD_DUMPED; self.si_status = code & 0x7f.

### Out of Scope

- `wait4(2)` (Tier-5 separate doc — legacy variant returning waitstatus int).
- `pidfd_open(2)` (Tier-5 separate doc — pidfd allocator).
- `clone3(2) CLONE_PIDFD` (covered in `clone3.md`).
- LSM `task_wait` hook (Tier-3 in `security/lsm-hooks.md`).
- Process-tree reaper / subreaper (`prctl(PR_SET_CHILD_SUBREAPER)`) (Tier-3 in `kernel/exit.md`).
- Implementation code.

### signature

```c
int waitid(idtype_t idtype, id_t id, siginfo_t *infop,
           int options, struct rusage *rusage);
```

(Glibc-level `waitid(3)` omits the trailing `rusage`; the raw syscall always carries it. Pass `NULL` to ignore.)

### parameters

| Param | Type | Direction | Semantics |
|---|---|---|---|
| `idtype` | `idtype_t` (`int`) | in | Selector domain: `P_ALL`, `P_PID`, `P_PGID`, `P_PIDFD`. |
| `id` | `id_t` (`unsigned int`) | in | Domain-specific identifier (ignored for `P_ALL`). |
| `infop` | `siginfo_t __user *` | out | Filled with reaped child's siginfo, OR zeroed if no child and `WNOHANG`. May be `NULL`. |
| `options` | `int` | in | Bitwise OR of `WNOHANG`, `WUNTRACED` (alias `WSTOPPED`), `WCONTINUED`, `WEXITED`, `WNOWAIT`. At least one of `WEXITED/WSTOPPED/WCONTINUED` MUST be set. |
| `rusage` | `struct rusage __user *` | out | Optional resource-accounting destination. May be `NULL`. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success: either a child event was reaped, OR `WNOHANG` was set and no eligible child was immediately reapable (`infop->si_pid == 0` distinguishes). |
| `-1` | Error; `errno` set as below. The libc wrapper translates positive kernel errors. |

### errors

| `errno` | Cause |
|---|---|
| `ECHILD` | No child matches `idtype/id`, OR there is an unrelated process group / pidfd target. |
| `EINTR` | Caller blocked and was interrupted by an unmasked signal (no `SA_RESTART`). |
| `EINVAL` | `idtype` is not `P_ALL/PID/PGID/PIDFD`; `options` has invalid bits or none of `WEXITED/WSTOPPED/WCONTINUED`; `P_PIDFD` with `WNOWAIT` on certain kernels. |
| `EFAULT` | `infop` or `rusage` is non-NULL and not in caller's writable address space. |
| `EBADF` | `idtype == P_PIDFD` and `id` is not an open file descriptor. |
| `EPERM` | LSM (SELinux, AppArmor) denied observation of the target task. |
| `EAGAIN` | Reserved; not normally returned (`WNOHANG` returns 0 instead). |

### abi surface

```text
__NR_waitid (x86_64)    = 247
__NR_waitid (i386)      = 284
__NR_waitid (arm64)     = 95   (generic-syscall)
__NR_waitid (generic)   = 95

typedef enum {
    P_ALL   = 0,
    P_PID   = 1,
    P_PGID  = 2,
    P_PIDFD = 3,
} idtype_t;

/* options bits */
#define WNOHANG       0x00000001
#define WUNTRACED     0x00000002
#define WSTOPPED      WUNTRACED
#define WEXITED       0x00000004
#define WCONTINUED    0x00000008
#define WNOWAIT       0x01000000
#define __WCLONE      0x80000000   /* legacy clone-thread (illegal w/ waitid) */
#define __WALL        0x40000000   /* legacy wait-all (illegal w/ waitid) */
#define __WNOTHREAD   0x20000000   /* legacy no-cross-thread (illegal w/ waitid) */

/* siginfo_t fields populated on success */
si_signo  = SIGCHLD
si_errno  = 0
si_code   = CLD_EXITED | CLD_KILLED | CLD_DUMPED | CLD_TRAPPED |
            CLD_STOPPED | CLD_CONTINUED
si_pid    = reaped child's TGID in caller's PID-namespace, or 0 (WNOHANG no-event)
si_uid    = child's real UID
si_status = exit code (CLD_EXITED) OR signal number (KILLED/DUMPED/STOPPED/...)
```

### compatibility contract

REQ-1: Syscall number is **247** on x86_64; **95** on arm64/generic; **284** on i386.

REQ-2: `idtype` MUST be exactly one of `P_ALL` (id ignored), `P_PID` (`id` = TGID), `P_PGID` (`id` = pgid; `id == 0` means caller's pgid), `P_PIDFD` (`id` = pidfd, a non-negative fd). Anything else: `EINVAL`.

REQ-3: `options` MUST contain at least one of `WEXITED`, `WSTOPPED`, `WCONTINUED`; otherwise `EINVAL`. Legacy `__WCLONE/__WALL/__WNOTHREAD` bits are rejected.

REQ-4: `WNOHANG` semantics: if no eligible child is immediately reapable, return `0` and zero `infop` (specifically `si_pid = 0`, `si_signo = 0`). The kernel writes a full zeroed `siginfo_t`; userspace MUST check `si_pid == 0` (NOT only the return value) to detect this case.

REQ-5: `WNOWAIT` semantics: child event observable WITHOUT consuming; the child remains in zombie / stopped / continued state. Subsequent `waitid` without `WNOWAIT`, or `wait4`, reaps definitively. Required for ptrace cooperation.

REQ-6: `WEXITED` selects exited children (zombies). `WSTOPPED/WUNTRACED` selects stopped (SIGSTOP/job-control). `WCONTINUED` selects continued (SIGCONT delivered while job-controlled). Each is independent.

REQ-7: `P_PIDFD` requires `id` reference an open pidfd to a TGID-leader task. Threads' pidfds: `EINVAL`. Caller need not be a parent; ANY pidfd-holder can `waitid(WNOWAIT|WEXITED)` to observe — but only the parent or `subreaper` may reap definitively (kernel returns `ECHILD` to non-parents without `WNOWAIT`).

REQ-8: `siginfo_t` MUST be cleared to zero by the kernel before any field is written, to prevent stale-stack info-leak. `si_signo` is unconditionally `SIGCHLD`; `si_code` ∈ {`CLD_EXITED`, `CLD_KILLED`, `CLD_DUMPED`, `CLD_TRAPPED`, `CLD_STOPPED`, `CLD_CONTINUED`}.

REQ-9: `si_pid` is rendered in the caller's active PID-namespace. A reaper in the host namespace sees the inner task's outer-ns PID; a reaper in the inner ns sees its own ns view.

REQ-10: `rusage` is non-cumulative for the reaped child only (`RUSAGE_CHILDREN` is the cumulative aggregate elsewhere). Optional: `NULL` skips the copy.

REQ-11: PaX UDEREF: copy_to_user(`infop`, `rusage`) MUST pass UDEREF unprivileged-dereference checks before any kernel-side fields are written. Pre-validate that both pointers, if non-NULL, lie in caller's writable userspace.

REQ-12: `EINTR` MUST be returned when the calling thread blocks in `do_wait` and is interrupted by an unmasked, non-`SA_RESTART` signal. With `SA_RESTART`, the syscall transparently restarts via `ERESTARTSYS`.

REQ-13: `ECHILD` MUST be returned if the calling task has no children matching `idtype/id` AND no descendant subreaper-eligible tasks. `WNOHANG` does not change this: lack-of-eligibility still yields `ECHILD`.

REQ-14: LSM hook `security_task_wait(task)` is invoked per candidate task (called from `wait_consider_task`); SELinux may deny observation.

REQ-15: `seccomp` filters see the syscall args verbatim, including raw `idtype` integer and pointers. `seccomp` BPF MAY block specific `idtype` values (e.g. forbid `P_PIDFD` for an unprivileged sandbox).

REQ-16: After successful reaping (no `WNOWAIT`), the child's `task_struct` is released via `release_task()`; the child becomes unreachable; subsequent `waitid` for that PID returns `ECHILD` (unless PID reused — pidfds prevent this).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `options_validated` | INVARIANT | per-waitid: reject if no wait-flag set or any unknown bit present. |
| `idtype_in_range` | INVARIANT | per-waitid: idtype ∈ {P_ALL, P_PID, P_PGID, P_PIDFD}. |
| `siginfo_zeroed_before_write` | INVARIANT | per-success: siginfo cleared before populate. |
| `uderef_pre_copy_to_user` | INVARIANT | per-copy: access_ok_uderef passes before write. |
| `wnowait_does_not_release_task` | INVARIANT | per-WNOWAIT: child remains in zombie/stopped/continued. |
| `pidfd_refcount_balanced` | INVARIANT | per-P_PIDFD: fdget/fdput symmetric. |
| `pid_get_put_symmetric` | INVARIANT | per-P_PID/P_PGID: find_get_pid balanced with put_pid. |

### Layer 2: TLA+

`kernel/waitid.tla`:
- States: per-task exit_state ∈ {RUNNING, STOPPED, CONTINUED, ZOMBIE, RELEASED}; per-wait-queue pending_events.
- Properties:
  - `safety_no_double_reap` — per-child: at most one non-WNOWAIT consumer.
  - `safety_wnowait_idempotent` — per-WNOWAIT: state unchanged.
  - `safety_zombie_observable` — per-WEXITED: every EXIT-zombie is observable by parent.
  - `liveness_eventual_reap` — per-child: parent's blocking waitid eventually returns or EINTR.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_waitid` post: errno mapping correct | `Waitid::do_waitid` |
| `consider` post: si_pid in caller's ns | `Waitid::consider` |
| `populate_exit` post: si_code valid CLD_* | `SigInfo::populate_exit` |
| pidfd-bound: thread-pidfd rejected | `P_PIDFD` branch |

### Layer 4: Verus / Creusot functional

Per-`waitid(2)` man-page equivalence. LTP `waitid01..waitid11` pass. POSIX.1-2008 `waitid` semantics verified. `pidfd_open + waitid(P_PIDFD)` round-trip race-free.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`waitid(2)` reinforcement:

- **Per-options-bitmask validation strict** — defense against per-undefined-bit forward-compat hazards.
- **Per-siginfo full-zero before populate** — defense against per-stale-kernel-stack info-leak (CVE-2014-1737-class).
- **Per-pidfd validity gate** — defense against per-PID-reuse confusion when supervisors reap via PID.
- **Per-LSM task-wait hook** — defense against per-cross-domain observation.
- **Per-pidns vnr translation** — defense against per-ns leak of host PIDs into containers.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `siginfo_t *infop`** — every copy_to_user(`infop`, &info) MUST traverse UDEREF SMAP/SMEP gates; raw kernel pointers cannot bleed into userspace via misaligned/incomplete writes. UDEREF rejects non-userspace targets BEFORE the kernel populates si_code/si_pid.
- **PaX UDEREF on `struct rusage *rusage`** — same check on rusage pointer pre-write; rejects partial writes that could leak kernel memory layout.
- **PaX UDEREF on `sigset_t`-adjacent** — not applicable to waitid (no sigset arg), but the siginfo path inherits UDEREF gating from the shared copy_to_user infrastructure.
- **GRKERNSEC_HARDEN_PTRACE process-isolation** — `P_PIDFD` waiters on a process they do not own (non-parent, non-CAP_SYS_PTRACE) are restricted to `WNOWAIT`; only the parent or a CAP_SYS_PTRACE holder may definitively reap. Stops "steal-the-exit-status" attacks against parents in another security domain.
- **PIDFD_SELF safe self-target** — `waitid(P_PIDFD, PIDFD_SELF, ...)` returns `EINVAL` (you cannot reap yourself); grsec explicitly fast-rejects to preempt confused-deputy waits.
- **GRKERNSEC_PROC filtered si_pid** — when caller's `/proc/<pid>` visibility is restricted by GRKERNSEC_PROC, the returned `si_pid` in siginfo is filtered to 0 if the reapee would otherwise be invisible to the reaper. (Only applies to non-parent observers via `WNOWAIT`; the parent always sees its child.)
- **GRKERNSEC_AUDIT_GROUP** — repeated `WNOHANG` polls from a marked group are rate-limited / audited to detect process-reaping reconnaissance.
- **No_new_privs neutral** — NNP does not change waitid semantics; child reaping is unprivileged for direct parents by POSIX mandate.
- **Anti-fingerprint hardening** — `si_uid` of reaped child is filtered through namespace-mapped uid_t (per-PID-ns), not host kernel uid_t, preventing host-side uid disclosure to containerized reapers.
- **Capability gate for cross-domain WNOWAIT** — observing via `P_PIDFD | WNOWAIT` of a non-child process requires CAP_SYS_PTRACE in grsec hardened mode (upstream allows any pidfd-holder; grsec tightens this to reduce side-channel observation of foreign processes).
- **Seccomp interaction** — seccomp filters may forbid `P_PIDFD` for unprivileged sandboxes; grsec recommends a default-deny posture with `P_PID|P_PGID|P_ALL` only.

