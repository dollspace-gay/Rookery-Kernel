# Tier-5 syscall: rt_sigqueueinfo(2) — syscall 129

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/signal.c (sys_rt_sigqueueinfo, do_rt_sigqueueinfo, __copy_siginfo_from_user)
  - include/linux/signal.h (send_sig_info, group_send_sig_info)
  - include/uapi/asm-generic/siginfo.h (siginfo_t, SI_USER, SI_QUEUE, SI_TKILL)
  - arch/x86/entry/syscalls/syscall_64.tbl (129  common  rt_sigqueueinfo)
-->

## Summary

`rt_sigqueueinfo(2)` queues a realtime signal to a process (thread
group) together with a caller-supplied `siginfo_t` payload. Unlike
`kill(2)`, which constructs `siginfo_t` server-side with `si_code =
SI_USER`, this syscall lets userspace attach an arbitrary
`union sigval` (`si_value`) — the canonical mechanism behind POSIX
`sigqueue(3)`. Realtime signals (32..64) queue multiply: every call
results in a distinct entry in the receiver's pending queue,
dequeued FIFO with siginfo intact.

Delivery is to ONE arbitrary thread of the target whose mask does
not block `sig`; if all threads have it masked, the signal queues
on the shared pending queue. Permission rules match `kill(2)`.
`sig == 0` performs the permission check only.

Critical for: glibc `sigqueue(3)`, POSIX timers' `SI_TIMER` payload,
POSIX message queues' `SI_MESGQ` payload, container-aware
signal-passing where uid mapping requires kernel-side translation.

This Tier-5 covers `kernel/signal.c::SYSCALL_DEFINE3(rt_sigqueueinfo,
...)` plus `do_rt_sigqueueinfo` and `__copy_siginfo_from_user`.

## Signature

```c
long rt_sigqueueinfo(pid_t tgid, int sig, siginfo_t *uinfo);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `tgid` | `pid_t` | in | Target thread group id (positive). Must be a TGID, not an arbitrary TID. |
| `sig` | `int` | in | Signal `0..SIGRTMAX (64)`. `0` = permission probe only. |
| `uinfo` | `siginfo_t *` | in | Caller-prepared siginfo. `si_signo == sig` and `si_code` in user-allowed set. |

## Return value

| Value | Meaning |
|---|---|
| `0`  | Signal queued (or `sig == 0` and permission OK). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `sig < 0` or `sig > _NSIG`; `uinfo.si_signo != sig`. |
| `EPERM`  | `uinfo.si_code` reserved (`SI_TKILL`, `SI_TIMER`, `SI_MESGQ`, etc.) for non-self target without `CAP_SYS_ADMIN`. |
| `EPERM`  | Caller lacks signal permission (uid mismatch + no `CAP_KILL`). |
| `ESRCH`  | No process with TGID == `tgid` exists. |
| `EAGAIN` | `RLIMIT_SIGPENDING` exceeded for caller's real-uid. |
| `EFAULT` | `uinfo` not accessible. |

## ABI surface

```text
__NR_rt_sigqueueinfo (x86_64)    = 129
__NR_rt_sigqueueinfo (i386)      = 178
__NR_rt_sigqueueinfo (generic)   = 138   /* arm64/riscv/loongarch */
__NR_rt_sigqueueinfo (powerpc)   = 177
__NR_rt_sigqueueinfo (s390x)     = 178
__NR_rt_sigqueueinfo (sparc)     = 106
```

### user-origin `si_code` allow-list

```text
SI_USER   (== 0)   only when tgid == caller.tgid (self-signal)
SI_QUEUE  (== -1)  unrestricted; canonical user-queue path
si_code <= 0 (other) — passed through, unless in
   {SI_TKILL, SI_TIMER, SI_MESGQ, SI_ASYNCIO, SI_SIGIO, SI_DETHREAD}
   which require CAP_SYS_ADMIN for non-self targets
si_code > 0 — kernel-reserved; rejected for non-self target
```

## Compatibility contract

REQ-1: Syscall number is **129** on x86_64; **138** on generic-syscall arches. ABI-stable.

REQ-2: `sig` validated against `[0, _NSIG]` (inclusive 64). Out-of-range → `-EINVAL`.

REQ-3: `uinfo` copied via `__copy_siginfo_from_user` (strict 128 bytes on LP64, UDEREF-protected, short-copy zero-extends on compat).

REQ-4: After copy, `uinfo.si_signo == sig`; mismatch → `-EINVAL`.

REQ-5: User-origin `si_code` allow-list check:
- Self target: permissive.
- Foreign target: `si_code <= 0` AND not in `{SI_TKILL, SI_TIMER, SI_MESGQ, SI_ASYNCIO, SI_SIGIO, SI_DETHREAD}` unless `CAP_SYS_ADMIN`. Else `-EPERM`.

REQ-6: Sender-identity overwrite (regardless of caller-supplied):
- `si_pid = current.tgid` (in target's pid_ns).
- `si_uid = current.cred.uid` (mapped through target's user_ns).

Prevents caller spoofing `si_pid`/`si_uid` to impersonate.

REQ-7: Permission: real or effective uid equals target's real or saved-set uid, OR `CAP_KILL` in target's user_ns. `SIGCONT` exception within same session.

REQ-8: Target lookup via `find_get_pid(tgid)` in caller's pid_ns. NULL → `-ESRCH`.

REQ-9: Realtime queueing: each call allocates a `struct sigqueue`, copies siginfo, links into `task.signal.shared_pending.list`. Bounded by `RLIMIT_SIGPENDING` per real-uid; over-limit → `-EAGAIN`.

REQ-10: Non-realtime signal: classical single-bit semantics. Second queue while bit set drops the siginfo (first retained).

REQ-11: Wakeup: select first eligible thread in target tgroup (sig not blocked); `signal_wake_up_state`. If none qualify, signal remains pending on shared queue.

REQ-12: Audit: `AUDIT_SYSCALL` captures sender pid, sender uid, target tgid, signal, sanitized si_code.

REQ-13: `sig == 0`: validate target + permission only; no enqueue, no wakeup.

REQ-14: Cross-pid-ns: `tgid` in caller's pid_ns. Receiver observes `si_pid` in receiver's pid_ns.

REQ-15: Cross-user-ns: `si_uid` mapped via `from_kuid_munged`; unmappable → `(uid_t)-1`.

REQ-16: Compat 32-bit: `compat_rt_sigqueueinfo` narrows padding; preserves allow-list.

REQ-17: `RLIMIT_SIGPENDING` enforced under `current.cred.user.lock`; no overflow into other users' slab.

## Acceptance Criteria

- [ ] AC-1: Syscall number 129 on x86_64; 138 on generic.
- [ ] AC-2: Send SIGRTMIN with `si_value.sival_int = 0xDEADBEEF`; peer's `rt_sigtimedwait` returns SIGRTMIN with matching `si_value`.
- [ ] AC-3: `si_signo` differs from `sig`: `-EINVAL`.
- [ ] AC-4: Foreign target, `si_code = SI_TKILL`: `-EPERM`.
- [ ] AC-5: Self target, any `si_code`: success.
- [ ] AC-6: Receiver observes `info.si_pid == caller.tgid` regardless of supplied value.
- [ ] AC-7: Receiver observes `info.si_uid == caller.uid` regardless of supplied value.
- [ ] AC-8: 1024 SIGRTMIN to peer (RLIMIT_SIGPENDING=1024): 1025th → `-EAGAIN`.
- [ ] AC-9: Non-existent tgid: `-ESRCH`.
- [ ] AC-10: Unprivileged cross-uid: `-EPERM`.
- [ ] AC-11: `sig == 0`, valid target, permission OK: returns 0; no signal queued.
- [ ] AC-12: Realtime signal queued 5 times: 5 distinct siginfos dequeued FIFO.
- [ ] AC-13: Non-realtime signal queued twice: second siginfo dropped.
- [ ] AC-14: `uinfo` faulting page: `-EFAULT`; no partial enqueue.

## Architecture

```rust
#[syscall(nr = 129, abi = "sysv")]
pub fn sys_rt_sigqueueinfo(
    tgid: pid_t,
    sig:  c_int,
    uinfo: UserPtr<UapiSiginfo>,
) -> isize {
    if sig < 0 || sig as usize > NSIG { return -EINVAL; }
    let mut info = UapiSiginfo::default();
    Siginfo::copy_from_user_strict(uinfo, &mut info)?;
    if info.si_signo != sig { return -EINVAL; }

    let target_is_self = tgid == Task::current().tgid;
    Signal::check_user_si_code(&info, target_is_self)?;

    info.si_pid = Task::current().tgid_in(Task::pid_ns_of(tgid));
    info.si_uid = Cred::current().uid.in_ns(Task::userns_of(tgid));

    Signal::do_rt_sigqueueinfo(tgid, sig, &info)?;
    0
}
```

`Signal::do_rt_sigqueueinfo`: lookup pid → `check_kill_permission` → if `sig == 0` return Ok → `group_send_sig_info` (RLIMIT-checked enqueue under siglock) → `signal_wake_up_state`.

`Signal::check_user_si_code`: if self → Ok; if `si_code >= 0` → `-EPERM`; if in restricted set without `CAP_SYS_ADMIN` → `-EPERM`; else Ok.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sig_range_strict` | INVARIANT | sig ∈ [0, NSIG] OR -EINVAL. |
| `si_signo_matches_sig` | INVARIANT | info.si_signo != sig ⟹ -EINVAL. |
| `user_si_code_allow_list` | INVARIANT | non-self ⟹ si_code in allow-list OR -EPERM. |
| `sender_identity_overwritten` | INVARIANT | si_pid/si_uid set kernel-authoritatively before enqueue. |
| `siginfo_copy_bounds` | INVARIANT | strict 128-byte copy on LP64. |
| `rlimit_sigpending_enforced` | INVARIANT | queue full ⟹ -EAGAIN. |

### Layer 2: TLA+

`kernel/rt_sigqueueinfo.tla`:
- States: VALIDATE_SIG → COPY_SIGINFO → CHECK_SI_SIGNO → CHECK_SI_CODE → OVERWRITE_SENDER_ID → LOOKUP_TGID → CHECK_PERMISSION → ENQUEUE → WAKE.
- Properties:
  - `safety_si_pid_authentic`, `safety_si_uid_authentic` — receiver always observes kernel-authoritative values.
  - `safety_no_spoofed_si_code` — receiver never sees a kernel-reserved si_code from unprivileged sender.
  - `safety_rlimit_bounded` — slab allocation ≤ RLIMIT_SIGPENDING per uid.
  - `liveness_rt_signal_delivered` — every queued rt signal eventually dequeued.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_rt_sigqueueinfo` post: 0 ⟹ exactly one sigqueue entry enqueued | `sys_rt_sigqueueinfo` |
| `check_user_si_code` post: Ok ⟹ allow-list satisfied OR self target | `check_user_si_code` |
| `copy_siginfo_from_user` post: 128 bytes copied; padding zeroed | `Siginfo::copy_from_user_strict` |

### Layer 4: Verus / Creusot functional

POSIX `sigqueue(3)` semantic equivalence. LTP `rt_sigqueueinfo01..03`, glibc `signal/tst-sigqueue*`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

- **Per-strict `si_signo == sig` check** — defense against per-mismatch signal smuggling.
- **Per-user-origin si_code allow-list** — defense against per-spoof of `SI_TIMER`/`SI_TKILL`/`SI_MESGQ` which other kernel paths trust.
- **Per-sender identity overwrite** — defense against per-spoof of `si_pid`/`si_uid`.
- **Per-RLIMIT_SIGPENDING strict** — defense against per-slab-exhaustion via realtime-signal flood.
- **Per-permission via real/saved uid OR CAP_KILL** — defense against per-cross-uid injection.
- **Per-pid_ns translation for tgid lookup** — defense against per-namespace pid confusion.
- **Per-user_ns mapping for si_uid** — defense against per-container uid leakage.

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at syscall entry** — randomizes kernel-stack layout per call; combined with per-call `UapiSiginfo` zero-init, defeats per-stack-leak via siginfo padding side-channel.
- **PaX UDEREF on `siginfo_t`** — `copy_siginfo_from_user` runs with SMAP/PAN + UDEREF; pointer mistakes cannot silently dereference user space.
- **PaX MEMORY_SANITIZE** — local `UapiSiginfo` zeroed before strict copy; padding never leaks per-stack/heap bytes into the receiver.
- **GRKERNSEC_SIGNALS SI_USER strict-clone** — caller-supplied `si_code = SI_USER` to a foreign target is REJECTED; SI_USER is reserved for kernel-synthesized `kill(2)`. The allow-list at REQ-5 enforces this; grsec makes it mandatory + audited.
- **GRKERNSEC_SIGNALS SI_TKILL / SI_QUEUE allow-list** — only `SI_QUEUE` is freely allowed to non-self; `SI_TKILL`/`SI_TIMER`/`SI_MESGQ`/`SI_ASYNCIO`/`SI_SIGIO`/`SI_DETHREAD` gated by `CAP_SYS_ADMIN`. Caller cannot spoof a timer-fire or thread-directed kill to a foreign process.
- **GRKERNSEC_BRUTE on signal-flood** — repeated cross-uid `rt_sigqueueinfo` probing for races trips the per-uid brute-force counter.
- **PaX KERNEXEC** — `do_rt_sigqueueinfo` in read-only-after-init kernel text; cannot be patched to skip allow-list or sender-id overwrite.
- **GRKERNSEC_HARDEN_PTRACE** — tracer cannot inject via `rt_sigqueueinfo` across credential boundaries; cross-cred ptrace check fails before enqueue.
- **GRKERNSEC_PROC restrictions** — sender pid/uid (visible to receiver) respects per-uid hide-filter; unmappable mapping → overflow uid.
- **PaX TASK_HARDENING** — sigqueue slab allocation audited under `RLIMIT_SIGPENDING`; over-limit is hard `-EAGAIN` with no kmalloc retry escalation, defeating per-slab-exhaustion DoS.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `rt_tgsigqueueinfo(2)` (separate Tier-5 — thread-directed variant).
- `rt_sigaction(2)` / `rt_sigprocmask(2)` / `rt_sigtimedwait(2)` / `rt_sigsuspend(2)` / `rt_sigpending(2)` (separate Tier-5 docs).
- `kill(2)` / `tkill(2)` / `tgkill(2)` (separate Tier-5 docs).
- Signal-delivery internals (`send_signal`, `group_send_sig_info`, `complete_signal` — Tier-3).
- Compat 32-bit variants (Tier-3).
- POSIX timer + mqueue siginfo paths (Tier-3).
- Implementation code.
