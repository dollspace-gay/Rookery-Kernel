# Tier-5 syscall: kill(2) — syscall 62

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/signal.c (sys_kill, kill_something_info, kill_pid_info)
  - include/linux/signal.h (struct k_sigaction)
  - include/uapi/asm-generic/siginfo.h (siginfo_t, SI_USER)
  - arch/x86/entry/syscalls/syscall_64.tbl (62  common  kill)
-->

## Summary

`kill(2)` sends a signal to a process or process group. Despite the
name, it does not necessarily terminate — the action depends on the
target's signal disposition: a handler runs, the default action
(terminate, core-dump, stop, continue, or ignore) executes, the signal
is masked (queued until unblocked), or the signal is ignored entirely.

The target is selected by `pid` with the following dispatch:
- `pid > 0`: signal the process with TGID = `pid`. Delivered to one
  arbitrary thread of that thread group whose mask does not block the
  signal.
- `pid == 0`: signal every process in the caller's process group.
- `pid < -1`: signal every process in the process group `-pid`.
- `pid == -1`: signal every process the caller has permission to signal
  EXCEPT init (pid 1) and the caller itself.

The signal carries a kernel-constructed `siginfo_t` with `si_code =
SI_USER`, `si_pid = caller.pid` (in caller's pid_ns), `si_uid =
caller.real_cred.uid`. The receiver can inspect these to identify the
sender via `SA_SIGINFO` handler.

Permission rules (POSIX):
- The caller may signal the target iff:
  - Caller has `CAP_KILL` in the target's user-namespace, OR
  - Caller's real or effective uid equals the target's real or
    saved-set uid.
- Exception for `SIGCONT`: also permitted within the same session,
  irrespective of uid match.

Critical for: every shell `kill` builtin, every supervisor (`systemd`,
`runit`, `s6`) terminating services, every test framework signaling
workers, every signal-based IPC (`pthread_kill` uses `tgkill`, NOT
`kill`).

This Tier-5 covers `kernel/signal.c::SYSCALL_DEFINE2(kill, ...)` plus its
`kill_something_info` / `kill_pid_info` workhorses (~80 lines of entry
surface). The full delivery + handler-invocation path is Tier-3 covered
in `kernel/signal.c`.

## Signature

```c
long kill(pid_t pid, int sig);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pid` | `pid_t` | in | Target selector. Positive = TGID of process. Zero = caller's process group. Negative-but-not-`-1` = `-pid` is a process group GID. `-1` = broadcast (caller-permitted). |
| `sig` | `int` | in | Signal number `0..SIGRTMAX (64)`. `sig == 0` is the special "permission check" — no signal queued, but error semantics apply. |

## Return value

| Value | Meaning |
|---|---|
| `0`  | Success: signal queued (or `sig == 0` and permission granted). |
| `-1` + `errno` | Failure (no signal delivered). |

For `pid == -1` and `pid < -1` (broadcast): returns `0` if AT LEAST ONE
target was successfully signaled; `-ESRCH` if no targets matched; the
first encountered permission error otherwise.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `sig < 0` or `sig > _NSIG (64)`. |
| `EPERM`  | Caller lacks permission to signal the target. |
| `ESRCH`  | No process or process group matches `pid`. |
| `ESRCH`  | Target task is in `EXIT_DEAD` (already reaped) — race window. |

## ABI surface

```text
__NR_kill (x86_64)    = 62
__NR_kill (i386)      = 37
__NR_kill (generic)   = 129   /* arm64/riscv/loongarch */
__NR_kill (powerpc)   = 37
__NR_kill (s390x)     = 37
__NR_kill (sparc)     = 37
__NR_kill (mips O32)  = 4037
__NR_kill (mips N64)  = 5060
__NR_kill (alpha)     = 37
__NR_kill (parisc)    = 37
```

### Signal numbers

(See `uapi/headers/signal.md` for the full table — 1..31 standard, 32..64 RT.)

### `siginfo_t` constructed by kill(2)

```text
siginfo_t {
    si_signo  = sig,
    si_errno  = 0,
    si_code   = SI_USER,                     // -1: from kill(2) / sigqueue(2)-no-value
    si_pid    = current.pid (in caller pid_ns),
    si_uid    = current.real_cred.uid,
    // sig-specific arm fields are zero
}
```

`SI_USER = 0` in mainline Linux (despite the asm-generic header
declaring it negative in some arches' compat path — the value queued
is always 0 for the `SI_USER` code). Receiver-side discrimination via
`si_code` reliably identifies sender-initiated signals.

## Compatibility contract

REQ-1: The syscall number is **62** on x86_64, **129** on
generic-syscall archs. ABI-stable forever.

REQ-2: `sig` MUST be in `[0, _NSIG (64)]`. Out-of-range returns
`-EINVAL`. `sig == 0` is permitted and means "permission probe only".

REQ-3: `pid > 0`: target is the thread group with `tgid == pid` in the
caller's pid_namespace. Signal is delivered to the thread group
collectively; one thread (chosen by `__send_signal_locked` heuristics —
typically the group leader if it doesn't block, otherwise any unblocking
thread) runs the action.

REQ-4: `pid == 0`: target is every process in `current.pgrp` (caller's
process group). The signal MUST be queued to each member that the
caller has permission to signal; lack of permission for ONE member does
NOT prevent delivery to others; if no member receives the signal,
return `-EPERM`.

REQ-5: `pid < -1`: target is every process in the process group
`-pid`. Same rules as `pid == 0` but with explicit pgrp ID.

REQ-6: `pid == -1`: target is every process for which the caller has
permission, EXCLUDING:
- pid 1 (init).
- The caller itself.
- Kernel threads (PF_KTHREAD).

Returns `0` if any target was signaled, `-ESRCH` if no eligible target
exists, `-EPERM` if all eligible targets denied permission.

REQ-7: Permission check (`check_kill_permission`):
- Allow if `current` has `CAP_KILL` in the target's user_namespace.
- Else allow if `current.real_cred.uid == target.real_cred.uid` OR
  `current.euid == target.real_cred.uid` OR
  `current.real_cred.uid == target.suid` OR
  `current.euid == target.suid`.
- Special-case `SIGCONT`: also allow if `current.session ==
  target.session` regardless of uid.

REQ-8: `sig == 0` (permission probe): same permission check as a real
signal. Returns `0` if permitted, `-EPERM` / `-ESRCH` as appropriate.
No `siginfo` queued.

REQ-9: Target in `EXIT_DEAD` (zombie reaped): treated as non-existent —
returns `-ESRCH`.

REQ-10: Target with `SIGNAL_UNKILLABLE` (init in pid_ns root) and signal
in `{SIGKILL, SIGSTOP, SIGTERM}`: silently dropped (delivered = 0; ret
0). Per-pid-ns init protection.

REQ-11: SIGRT signals (32..64): queued individually (NOT coalesced).
Per-uid `RLIMIT_SIGPENDING` enforced on queue length.

REQ-12: Standard signals (1..31): coalesced — if the signal is already
pending, the new instance does NOT enqueue. `siginfo` of the first
queued instance is preserved.

REQ-13: `si_pid` and `si_uid` are recorded in the CALLER's pid_ns and
user_ns at queue time. If the receiver is in a different pid_ns,
`si_pid` is translated to the receiver's pid_ns (showing 0 if the
caller has no visible pid).

REQ-14: pid_namespace boundary:
- Caller in init pid_ns: can signal any pid (subject to credential
  rules).
- Caller in nested pid_ns: can signal only pids visible in that pid_ns.
- A nested-pid_ns caller signaling pid 1 (the namespace's init):
  treated specially with `SIGNAL_UNKILLABLE` protection.

REQ-15: Multithreaded target: signal is queued on `signal->shared_pending`
and one thread is chosen. Thread choice prefers the thread group leader
if it doesn't have the signal blocked; otherwise any unblocking thread.
If all threads block: signal remains queued on shared_pending until an
unblock.

REQ-16: Audit: AUDIT_KILL record with `target_pid`, `target_tgid`,
`sig`, and success/fail status.

REQ-17: SELinux/AppArmor hook: `security_task_kill(target, &info, sig,
cred)` consulted; may deny with `-EPERM` (or whatever LSM returns).

REQ-18: For `pid == -1`, the kernel walks `pidhash` or
`pid_namespace->idr` to enumerate; this is `O(N)` in number of tasks.
RCU read-side lock held during the walk.

REQ-19: `kill_something_info` dispatches:
- `pid > 0` → `kill_pid_info_type(sig, info, find_get_pid(pid), PIDTYPE_TGID)`.
- `pid == 0` → `__kill_pgrp_info(sig, info, task_pgrp(current))`.
- `pid < -1` → `__kill_pgrp_info(sig, info, find_vpid(-pid))`.
- `pid == -1` → broadcast loop.

REQ-20: Kernel-internal signals (e.g. `SIGSEGV` from a fault) bypass
`kill()` and call `force_sig_info` directly with `si_code` set to a
kernel code (e.g. `SEGV_MAPERR`); these are NOT spoofable via `kill(2)`
because the kernel writes `SI_USER` for kill-delivered signals.

## Acceptance Criteria

- [ ] AC-1: Syscall number is 62 on x86_64; 129 on generic.
- [ ] AC-2: `kill(pid, SIGTERM)` where pid is a child of caller: SIGTERM delivered with `si_code = SI_USER`, `si_pid = caller.pid`.
- [ ] AC-3: `kill(pid, 0)` permission probe: returns 0 if permitted, `-EPERM` / `-ESRCH` otherwise.
- [ ] AC-4: `kill(0, SIGTERM)`: every process in caller's pgrp receives SIGTERM.
- [ ] AC-5: `kill(-100, SIGTERM)` where 100 is a pgrp id: every process in pgrp 100 receives SIGTERM.
- [ ] AC-6: `kill(-1, SIGTERM)` from root: every non-kthread process except init and self receives SIGTERM.
- [ ] AC-7: `kill(1, SIGKILL)` from non-root (init pid_ns): returns 0 but signal is silently dropped (init unkillable).
- [ ] AC-8: `kill(other_user_pid, SIGTERM)` from non-CAP_KILL caller with mismatched uid → `-EPERM`.
- [ ] AC-9: `kill(self_pid, SIGCONT)` to a same-session process: succeeds regardless of uid.
- [ ] AC-10: `kill(self_pid, -1)` → `-EINVAL`.
- [ ] AC-11: `kill(self_pid, 65)` → `-EINVAL`.
- [ ] AC-12: `kill(nonexistent_pid, SIGTERM)` → `-ESRCH`.
- [ ] AC-13: `kill(self_pid, SIGUSR1)` repeatedly: standard signal coalesced (only one queued); SIGRT signals queued individually.
- [ ] AC-14: `kill(child_in_nested_pidns_pid, SIGTERM)`: si_pid in child sees caller's pid translated to child's pid_ns.
- [ ] AC-15: Auditd records target_pid, target_tgid, sig, success.
- [ ] AC-16: Concurrent kill + exit race on target: kernel reports `-ESRCH` after target enters EXIT_DEAD.
- [ ] AC-17: `RLIMIT_SIGPENDING` exceeded for SIGRT to a target: returns `-EAGAIN`.
- [ ] AC-18: LSM hook denies: returns whatever the LSM returns (typically `-EACCES`).

## Architecture

```rust
#[syscall(nr = 62, abi = "sysv")]
pub fn sys_kill(pid: i32, sig: i32) -> isize {
    if sig < 0 || sig > _NSIG as i32 { return -EINVAL; }

    // Build kernel siginfo
    let info = KernelSiginfo {
        si_signo: sig,
        si_errno: 0,
        si_code:  SI_USER,
        si_pid:   Task::current().pid_in(ns_of_pid(pid_for_dispatch(pid))),
        si_uid:   Task::current().real_cred.uid,
        ..Default::default()
    };

    Signal::kill_something_info(sig, &info, pid)
}
```

`Signal::kill_something_info(sig, info, pid) -> isize`:
1. match pid {
2.   p if p > 0 => {
3.     let target_pid = PidNs::current().find_vpid(p as u32);
4.     match target_pid { None => Err(-ESRCH), Some(pid) => Signal::kill_pid_info(sig, info, pid, PIDTYPE_TGID) }
5.   }
6.   0 => Signal::kill_pgrp_info(sig, info, Task::current().pgrp()),
7.   p if p < -1 => Signal::kill_pgrp_info(sig, info, PidNs::current().find_vpid((-p) as u32)),
8.   -1 => Signal::kill_broadcast(sig, info),
9. }

`Signal::kill_pid_info(sig, info, pid, type) -> isize`:
1. let task = pid.task(type);   // RCU-protected
2. if task.is_none() { return -ESRCH; }
3. let task = task.unwrap();
4. /* SIGNAL_UNKILLABLE */
5. if task.signal.flags & SIGNAL_UNKILLABLE != 0 && fatal_sig(sig) { return 0; }
6. /* Permission */
7. Signal::check_kill_permission(sig, info, &task)?;
8. /* LSM */
9. Lsm::task_kill(&task, info, sig, current_cred())?;
10. /* Enqueue */
11. if sig == 0 { return 0; }
12. Signal::send_signal_locked(sig, info, &task, type)

`Signal::check_kill_permission(sig, info, target) -> Result<()>`:
1. if Capabilities::has_capability_ns(CAP_KILL, target.user_ns) { return Ok(()); }
2. let c = current_cred(); let t = target.real_cred;
3. if c.uid == t.uid || c.euid == t.uid || c.uid == t.suid || c.euid == t.suid { return Ok(()); }
4. if sig == SIGCONT && current.session == target.session { return Ok(()); }
5. Err(-EPERM)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sig_range` | TOTAL | sig ∉ [0,_NSIG] ⟹ -EINVAL. |
| `siginfo_si_user_set` | INVARIANT | si_code == SI_USER, si_pid/uid kernel-credentialed. |
| `pid_dispatch_total` | TOTAL | pid ∈ Z: dispatches to one of {pid>0, 0, pid<-1, -1} branches exhaustively. |
| `signal_unkillable_honored` | INVARIANT | SIGNAL_UNKILLABLE + fatal sig ⟹ ret 0, no enqueue. |
| `permission_checked_pre_enqueue` | INVARIANT | Permission denial occurs BEFORE any pending-queue mutation. |

### Layer 2: TLA+

`kernel/kill.tla`:
- States: VALIDATE → BUILD_SIGINFO → DISPATCH → (PER_PID | PER_PGRP | BROADCAST) → CHECK_PERM → MAYBE_ENQUEUE → RETURN.
- Concurrent: target exit on another CPU.
- Properties:
  - `safety_no_double_enqueue` — standard signal coalesces.
  - `safety_no_enqueue_on_perm_fail` — -EPERM ⟹ no shared_pending mutation.
  - `safety_si_user_immutable` — caller cannot set si_code to anything other than SI_USER via this path.
  - `liveness_eventual_delivery` — queued non-blocked signal eventually consumed by target.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_kill` post: si_user enforced; si_pid/uid kernel-credentialed | `sys_kill` |
| `kill_pid_info` post: returns 0 ⟺ enqueued (or sig==0 and permitted) | `Signal::kill_pid_info` |
| `check_kill_permission` total + sound | `Signal::check_kill_permission` |
| Broadcast (-1) excludes pid 1 + self + kthreads | `Signal::kill_broadcast` |

### Layer 4: Verus / Creusot functional

POSIX `kill(2)` semantic equivalence. LTP `kill01..kill09`,
`kselftests/sigaltstack/*` indirectly exercise. SIGRT queueing semantics
match `signal(7)`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`kill(2)` reinforcement:

- **Per-`SI_USER` siginfo kernel-built** — defense against per-spoofed-sender (cannot pretend to be kernel).
- **Per-si_pid translated through pid_ns** — defense against per-pid-leak across namespaces.
- **Per-permission check pre-enqueue** — defense against per-side-effect-on-deny.
- **Per-SIGNAL_UNKILLABLE init protection** — defense against per-init-kill catastrophe.
- **Per-RLIMIT_SIGPENDING SIGRT cap** — defense against per-RT-signal-flood DoS.
- **Per-coalescing standard signals** — defense against per-signal-storm queue explosion.
- **Per-LSM hook honored** — defense against per-MAC-policy bypass.
- **Per-audit AUDIT_KILL** — defense against per-attack-step elision.
- **Per-broadcast `-1` excludes pid 1, self, kthreads** — defense against per-broadcast self-DoS / init-kill / kthread-misdirect.
- **Per-`pid == 0` and `pid < -1` pgrp scoping** — defense against per-cross-session signal.

## Grsecurity / PaX surface

- **PAX_RANDKSTACK at syscall entry** — `sys_kill` randomizes kernel
  stack offset; the brief siglock window has per-call layout.
- **PaX UDEREF** — `sys_kill` has no user pointers; `pid` and `sig` are
  register-passed. Subsequent siginfo build is entirely kernel-side.
- **GRKERNSEC_HARDEN_PTRACE** — ptrace-injected signals via
  `PTRACE_KILL` / `PTRACE_INTERRUPT` honor the yama+grsec policy; a
  tracer cannot escalate kill rights through this path.
- **GRKERNSEC_PROC restrictions** — when caller is signaling a process
  whose `/proc/<pid>` is hidden by per-uid filter, the existence is
  leaked through `-ESRCH` vs `-EPERM`; Rookery merges both to `-ESRCH`
  under `grsec_proc_findtask=1` to deny the existence side-channel.
- **GRKERNSEC_CHROOT** — caller in chroot cannot signal a process
  outside the chroot subtree, even with matching uid (per
  `chroot_findtask`). Returns `-ESRCH`.
- **GRKERNSEC_BRUTE** — repeated `kill(target, SIGKILL)` attempts that
  fail with `-EPERM` (signal-spamming during privilege probe) increment
  per-uid brute-force counter; sustained → per-uid lockout.
- **GRKERNSEC_SIGNALS spoof prevention** — `si_code = SI_USER` is
  written by the kernel; userspace cannot forge `SI_KERNEL` /
  `SI_TKILL` / `SI_QUEUE` via this syscall (those codes are reserved
  for kernel-faults / tgkill / sigqueue, each with their own permission
  paths). Combined with si_pid / si_uid kernel-set, the receiver's
  `SA_SIGINFO` handler can trust the sender identity.
- **Per-grsec `chroot_caps`** — caller in chroot has bounded capset; if
  `CAP_KILL` is dropped on chroot entry (default policy), the
  uid-match fallback applies but no cross-uid signaling.
- **Per-grsec `gradm` learning** — RBAC may restrict the (subject,
  object, signal) triple — e.g. forbid SIGSTOP from non-debugger
  subjects.
- **Per-grsec `signal_logging`** — every `kill(2)` invocation by an
  unprivileged uid logged to dmesg / audit; `CAP_SYSLOG` gated read.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `tkill(2)` and `tgkill(2)` (separate Tier-5 docs).
- `rt_sigqueueinfo(2)` / `rt_tgsigqueueinfo(2)` (separate Tier-5 docs).
- `pidfd_send_signal(2)` (separate Tier-5 doc).
- Signal-delivery path (`kernel/signal.c::get_signal`, `do_signal` — Tier-3).
- `siginfo_t` complete layout (Tier-5 `uapi/headers/signal.md` — covered there).
- LSM `task_kill` hook implementations (Tier-3 LSM docs).
- Implementation code.
