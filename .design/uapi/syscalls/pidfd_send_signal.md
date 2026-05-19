# Tier-5 syscall: pidfd_send_signal(2) — syscall 424

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/signal.c (SYSCALL_DEFINE4(pidfd_send_signal))
  - kernel/signal.c (do_send_sig_info, group_send_sig_info)
  - kernel/pid.c (pidfd_get_pid)
  - include/uapi/asm-generic/siginfo.h
  - arch/x86/entry/syscalls/syscall_64.tbl (424 common pidfd_send_signal)
-->

## Summary

`pidfd_send_signal(2)` is the race-free `kill(2)` replacement: it sends a signal to a process identified by a **pidfd**, not by a PID integer. Because pidfds are stable references to a specific `struct pid` (created by `pidfd_open(2)` or `clone3(CLONE_PIDFD)`), the entire class of PID-reuse races that affects `kill(pid, sig)` is eliminated. The signal payload is described by an optional `siginfo_t` (mirroring `rt_sigqueueinfo(2)`); `NULL` produces a default kernel-synthesized `siginfo_t` equivalent to `kill(pid, sig)`. The `flags` argument is reserved (must be 0 in 5.3..5.18; 5.19+ accepts `PIDFD_SIGNAL_THREAD` and `PIDFD_SIGNAL_THREAD_GROUP` and `PIDFD_SIGNAL_PROCESS_GROUP`).

Critical for: container runtimes' graceful-shutdown plumbing (SIGTERM via pidfd, no PID race), service supervisors signaling watched processes, debugger SIGSTOP/SIGCONT control, sandbox escape termination, language-runtime SIGCHLD propagation, KVM guest-trigger via SIGSEGV, every "signal-a-specific-process-no-matter-what" pattern.

## Signature

```c
int pidfd_send_signal(int pidfd, int sig, siginfo_t *info, unsigned int flags);
```

## Parameters

| Param | Type | Direction | Semantics |
|---|---|---|---|
| `pidfd` | `int` | in | pidfd from `pidfd_open(2)` / `clone3(CLONE_PIDFD)`. |
| `sig` | `int` | in | Signal number (1..`_NSIG-1`). `0` permitted for existence check (no delivery). |
| `info` | `siginfo_t __user *` | in | Optional explicit siginfo; `NULL` ⇒ kernel synthesizes default (SI_USER). |
| `flags` | `unsigned int` | in | 5.3..5.18: must be `0`. 5.19+: bits below. |

5.19+ flags:
- `PIDFD_SIGNAL_THREAD`: deliver to the specific thread (if pidfd was opened with `PIDFD_THREAD`).
- `PIDFD_SIGNAL_THREAD_GROUP`: deliver to the thread group (default for TGID-leader pidfd).
- `PIDFD_SIGNAL_PROCESS_GROUP`: deliver to the process group (`killpg`-equivalent via pidfd).

## Return value

| Value | Meaning |
|---|---|
| `0` | Signal delivered (or existence-check succeeded if `sig == 0`). |
| `-1` | Error; `errno` set. |

## Errors

| `errno` | Cause |
|---|---|
| `EBADF` | `pidfd` is not a valid open fd. |
| `EINVAL` | `pidfd` is not a pidfd; `sig` is out of range; `flags` has invalid bits; `info.si_code` is kernel-reserved (SI_KERNEL or SI_TKILL with cross-process info). |
| `ESRCH` | Target has exited (pidfd refers to dead process). |
| `EPERM` | Caller lacks permission to send `sig` to target (cross-uid + no CAP_KILL). |
| `EFAULT` | `info` is non-NULL and not in caller's readable address space. |

## ABI surface

```text
__NR_pidfd_send_signal  (x86_64) = 424
__NR_pidfd_send_signal  (i386)   = 424
__NR_pidfd_send_signal  (arm64)  = 424
__NR_pidfd_send_signal  (generic)= 424

/* siginfo_t passed userspace: kernel-side struct kernel_siginfo, 128 bytes */
/* Layout identical to siginfo_t in include/uapi/asm-generic/siginfo.h */

/* Flags (5.19+): */
#define PIDFD_SIGNAL_THREAD          (1U << 0)
#define PIDFD_SIGNAL_THREAD_GROUP    (1U << 1)
#define PIDFD_SIGNAL_PROCESS_GROUP   (1U << 2)

/* Permitted info.si_code values for unprivileged callers: */
SI_USER     (0)        /* default */
SI_QUEUE    (-1)       /* sigqueue */
/* Kernel-reserved (caller must have CAP_KILL or be same-uid): */
SI_KERNEL, SI_TIMER, SI_MESGQ, SI_ASYNCIO, SI_TKILL, ...
```

## Compatibility contract

REQ-1: Syscall number is **424** on all architectures (uniformly assigned, added in 5.1).

REQ-2: `pidfd` MUST be a valid pidfd referring to a `struct pid` (created by `pidfd_open` or `clone3(CLONE_PIDFD)`). Any other fd type ⇒ `EINVAL`.

REQ-3: `sig` MUST be in `[0, _NSIG-1]` (`0..63` plus realtime signals). `sig == 0` is the existence check (no delivery). Out-of-range ⇒ `EINVAL`.

REQ-4: `info == NULL`: kernel synthesizes default siginfo:
- `si_signo = sig`
- `si_errno = 0`
- `si_code = SI_USER`
- `si_pid = task_tgid_vnr(current)`
- `si_uid = from_kuid(target_ns, current_uid())`

REQ-5: `info != NULL`: pre-validated via UDEREF; copied via `copy_from_user`. `info->si_signo` MUST equal `sig` argument. `si_code` MUST NOT be kernel-reserved (SI_KERNEL, SI_TIMER, SI_TKILL, etc.) unless caller has CAP_KILL or is sending to self.

REQ-6: Permission check: caller MUST be able to send `sig` to target per `kill(2)` rules: same-uid (real or effective), OR CAP_KILL, OR special `sig == SIGCONT` to a descendant in same session. Cross-namespace: target's PID must be visible.

REQ-7: `flags` (5.3..5.18): MUST be `0`; otherwise `EINVAL`. (5.19+): one-of-three target-scope flags; mutually exclusive.

REQ-8: Default (5.3..5.18, or no flags 5.19+): delivery scope matches the pidfd:
- TGID-leader pidfd: signal to thread group (group-level delivery; selected delivering thread).
- PIDFD_THREAD pidfd: signal to specific thread.

REQ-9: `PIDFD_SIGNAL_PROCESS_GROUP` (5.19+): delivers to the process group of the target (equivalent to `killpg(getpgid(target_pid), sig)`); the target's pgid is used.

REQ-10: After target exits, pidfd persists but `pidfd_send_signal` returns `ESRCH`. The pidfd remains a valid open fd until closed.

REQ-11: LSM hooks: `security_task_kill(target_task, info, sig, secid)` MUST fire per delivery; SELinux can deny.

REQ-12: PaX UDEREF on `info` userspace pointer if non-NULL.

REQ-13: `seccomp` filters see `pidfd, sig, info, flags`; SHOULD be allowed for sandboxed task managers; MAY restrict allowed `sig` values.

REQ-14: PID-namespace: target's `struct pid` is ns-agnostic at kernel level. The signal is delivered to the underlying task regardless of caller's ns view; however, permission checks use the caller's ns-mapped uid view.

REQ-15: `sig == SIGKILL` / `SIGSTOP`: unmaskable; bypass signal-mask but still subject to permission check.

REQ-16: Realtime signals (`SIGRTMIN..SIGRTMAX`): queued in target's per-process or per-thread queue; multiple deliveries are NOT merged (unlike standard signals which coalesce).

## Acceptance Criteria

- [ ] AC-1: `pidfd_send_signal(pidfd, SIGTERM, NULL, 0)` delivers SIGTERM to target.
- [ ] AC-2: `pidfd_send_signal(pidfd, 0, NULL, 0)` succeeds if target alive, else `ESRCH`.
- [ ] AC-3: `pidfd_send_signal(non_pidfd, ...)` returns `EINVAL`.
- [ ] AC-4: `pidfd_send_signal(pidfd, 999, NULL, 0)` returns `EINVAL` (sig out of range).
- [ ] AC-5: `pidfd_send_signal(pidfd_to_dead_process, SIGTERM, NULL, 0)` returns `ESRCH`.
- [ ] AC-6: Cross-uid signal without CAP_KILL returns `EPERM`.
- [ ] AC-7: `info != NULL` with `si_signo != sig` returns `EINVAL`.
- [ ] AC-8: `info.si_code = SI_KERNEL` from unprivileged caller returns `EPERM` (or `EINVAL`).
- [ ] AC-9: `info = bad-pointer` returns `EFAULT`.
- [ ] AC-10: `flags = 0xDEADBEEF` returns `EINVAL`.
- [ ] AC-11: `flags = PIDFD_SIGNAL_THREAD` on TGID-leader pidfd returns `EINVAL` (target mismatch, 5.19+).
- [ ] AC-12: Realtime signal (SIGRTMIN+0) queues and delivers multiple times.
- [ ] AC-13: Default siginfo: target sees `si_pid = caller_tgid` and `si_uid = caller_uid`.

## Architecture

```rust
#[syscall(nr = 424, abi = "sysv")]
pub fn sys_pidfd_send_signal(
    pidfd: i32,
    sig: i32,
    info: UserPtr<KernelSiginfo>,
    flags: u32,
) -> KResult<i32> {
    PidfdSendSignal::do_pidfd_send_signal(pidfd, sig, info, flags)
}
```

`PidfdSendSignal::do_pidfd_send_signal(pidfd, sig, info_uptr, flags) -> KResult<i32>`:
1. /* Validate sig */
2. if sig < 0 || sig >= _NSIG { return Err(EINVAL); }
3. /* Validate flags */
4. const ALLOWED: u32 = PIDFD_SIGNAL_THREAD | PIDFD_SIGNAL_THREAD_GROUP | PIDFD_SIGNAL_PROCESS_GROUP;
5. if flags & !ALLOWED != 0 { return Err(EINVAL); }
6. if (flags & ALLOWED).count_ones() > 1 { return Err(EINVAL); }   /* exclusive */
7. /* Resolve pidfd */
8. let file = fdget(pidfd)?;
9. let target_pid: &Arc<Pid> = PidfdOpen::pid_from_file(&file).ok_or(EINVAL)?;
10. let pidfd_flags = file.flags;
11. /* Determine target scope */
12. let scope = PidfdSendSignal::resolve_scope(flags, pidfd_flags);
13. /* Prepare siginfo */
14. let mut info = KernelSiginfo::zeroed();
15. if !info_uptr.is_null() {
    - access_ok_uderef(info_uptr, sizeof::<KernelSiginfo>())?;
    - copy_from_user(&mut info, info_uptr, sizeof::<KernelSiginfo>())?;
    - if info.si_signo != sig { return Err(EINVAL); }
    - /* Kernel-reserved si_code check */
    - if info.si_code >= 0 && info.si_code != SI_USER && info.si_code != SI_QUEUE {
      - if !capable(CAP_KILL) && current_uid() != target.real_uid { return Err(EPERM); }
    - }
16. } else {
    - info.si_signo = sig;
    - info.si_errno = 0;
    - info.si_code = SI_USER;
    - info.si_pid = task_tgid_vnr(current());
    - info.si_uid = from_kuid_caller_ns(current_uid());
17. }
18. /* Resolve task */
19. let task = target_pid.task_in(PIDTYPE_TGID).ok_or(ESRCH)?;
20. /* LSM */
21. security_task_kill(task, &info, sig, 0)?;
22. /* Permission check */
23. check_kill_permission(sig, &info, task)?;
24. /* Deliver per scope */
25. match scope {
    - Scope::Thread => do_send_sig_info(sig, &info, task, PIDTYPE_PID),
    - Scope::ThreadGroup => group_send_sig_info(sig, &info, task, PIDTYPE_TGID),
    - Scope::ProcessGroup => kill_pgrp_info(sig, &info, task_pgrp(task)),
26. }?;
27. Ok(0)

`PidfdSendSignal::resolve_scope(flags, pidfd_flags) -> Scope`:
1. if flags == 0 {
   - return if pidfd_flags & PIDFD_THREAD != 0 { Scope::Thread } else { Scope::ThreadGroup };
2. }
3. match flags {
   - PIDFD_SIGNAL_THREAD => Scope::Thread,
   - PIDFD_SIGNAL_THREAD_GROUP => Scope::ThreadGroup,
   - PIDFD_SIGNAL_PROCESS_GROUP => Scope::ProcessGroup,
   - _ => unreachable!(),
4. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sig_in_range` | INVARIANT | per-syscall: 0 ≤ sig < _NSIG. |
| `flags_validated` | INVARIANT | per-syscall: only allowed bits, mutually exclusive. |
| `siginfo_signo_matches` | INVARIANT | per-info: si_signo == sig argument. |
| `si_code_privileged_gate` | INVARIANT | per-kernel-reserved-si_code: CAP_KILL or same-uid. |
| `uderef_pre_copy_from_user` | INVARIANT | per-info: UDEREF before copy_from_user. |
| `pidfd_to_pid_resolved` | INVARIANT | per-pidfd: refers to struct pid. |
| `lsm_hook_fires` | INVARIANT | per-delivery: security_task_kill invoked. |

### Layer 2: TLA+

`kernel/pidfd-send-signal.tla`:
- States: per-task pending signal queue, per-process group set, per-pidfd target pid.
- Properties:
  - `safety_no_pid_reuse_delivery` — per-pidfd: signal delivered to original task or ESRCH.
  - `safety_permission_enforced` — per-delivery: caller has rights.
  - `safety_lsm_authorized` — per-delivery: LSM permitted.
  - `liveness_delivery_or_esrch` — per-call: terminates with delivery or ESRCH.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_pidfd_send_signal` post: returns 0 or specific errno | `PidfdSendSignal::do_pidfd_send_signal` |
| `resolve_scope` post: scope matches flags/pidfd_flags | `PidfdSendSignal::resolve_scope` |

### Layer 4: Verus / Creusot functional

Per-`pidfd_send_signal(2)` man-page equivalence. LTP `pidfd_send_signal01..04` pass. Round-trip with `pidfd_open + pidfd_send_signal(SIGTERM)` equivalent to `kill(pid, SIGTERM)` modulo PID-reuse race.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`pidfd_send_signal(2)` reinforcement:

- **Per-pidfd-stable target** — defense against per-PID-reuse race.
- **Per-si_code privilege gate** — defense against per-spoofed-kernel-siginfo.
- **Per-LSM task_kill hook** — defense against per-cross-domain signal.
- **Per-permission-check standard** — defense against per-cross-uid signal injection.
- **Per-UDEREF on siginfo** — defense against per-kernel-pointer info-leak.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `siginfo_t *info`** — every copy_from_user(info) MUST traverse UDEREF SMAP/SMEP gate; rejects kernel-half pointers before reading user-supplied siginfo. UDEREF is checked BEFORE any signal-delivery side effect.
- **PaX UDEREF on `sigset_t`-adjacent** — not directly applicable (no sigset arg); siginfo path inherits UDEREF gating.
- **GRKERNSEC_HARDEN_PTRACE process-isolation** — sending signals to non-ptrace-attachable processes via pidfd is denied in hardened mode (cross-uid + no CAP_KILL ⇒ EPERM, consistent with kill(2) but reinforced).
- **PIDFD_SELF safe self-target** — `pidfd_send_signal(pidfd_open(PIDFD_SELF), sig, ...)` is permitted unconditionally (self-signal is unprivileged); grsec fast-paths the self-target case to avoid permission round-trip.
- **GRKERNSEC_PROC restrictions** — when pidfd target is hidden from caller's /proc view, `pidfd_send_signal` returns ESRCH (consistent with kill(2) of hidden pids).
- **GRKERNSEC_AUDIT_GROUP** — pidfd_send_signal(SIGKILL) from marked groups is audited; unusual signal patterns (mass-kill via dup'd pidfds) flagged.
- **No_new_privs neutral** — NNP does not change pidfd_send_signal semantics; signal delivery is governed by uid+CAP_KILL.
- **Anti-fingerprint hardening** — default-synthesized siginfo.si_pid is rendered in TARGET's PID namespace, not caller's; prevents host-pid leakage to containerized signal-receivers.
- **Anti-spoof si_code check** — strict: SI_KERNEL / SI_TIMER / SI_TKILL / SI_ASYNCIO / SI_MESGQ from userspace require CAP_KILL OR same-uid; grsec tightens to CAP_KILL + same-cgroup for these kernel-reserved codes (upstream allows same-uid).
- **signalfd CLOEXEC mandatory** — when receiver uses signalfd to observe pidfd_send_signal-delivered signals, signalfd CLOEXEC default is enforced by grsec at signalfd create time.
- **Capability gate for SIGRTMIN broadcasts** — high-rate realtime signal flooding via pidfd is rate-limited; CAP_KILL required for >100Hz delivery in hardened mode (sysctl `kernel.pidfd_rtsig_rate_limit`).
- **PID-namespace strict mapping** — `info.si_pid` and `info.si_uid` rendered in the target's namespace, not the caller's; prevents the caller from forging cross-ns identity in user-supplied siginfo.
- **EPOLL fd-lifetime refcount strict** — pidfd held by epoll for exit-notification is properly ref-counted; signal-delivery via pidfd does not race with epoll cleanup.
- **Seccomp interaction** — seccomp filters MAY restrict `sig` argument to a whitelist (e.g., only SIGTERM/SIGKILL); default-deny hardening recommends per-uid signal-set policy.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `kill(2)` (Tier-5 separate doc — legacy PID-based signaling).
- `tkill(2)` / `tgkill(2)` (Tier-5 separate docs — thread-targeted variants).
- `rt_sigqueueinfo(2)` (Tier-5 separate doc — sigqueue with siginfo).
- `pidfd_open(2)` (Tier-5 separate doc).
- Signal delivery internals (Tier-3 in `kernel/signal.md`).
- Implementation code.
