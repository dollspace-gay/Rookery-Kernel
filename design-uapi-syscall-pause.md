---
title: "Tier-5 syscall: pause(2) — syscall 34"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`pause(2)` is the original UNIX "wait until I get a signal" primitive: it puts the calling task into `TASK_INTERRUPTIBLE` sleep and calls `schedule()`, blocking until **any** signal is delivered that either terminates the process or is caught by a registered signal handler. Once such a signal arrives, the handler (if any) is invoked and then `pause(2)` returns `-1` with `errno == EINTR`. There is no other return path: `pause(2)` never returns success. The syscall is functionally subsumed by `sigsuspend(2)` (which atomically sets a signal mask before blocking) and `pselect(2)` / `ppoll(2)` (which add timeouts), and is one of the simplest blocking syscalls in the kernel — about a dozen lines of source. Critical for: legacy daemons that block on `SIGUSR1` / `SIGHUP` reload triggers, init-script sleep-forever patterns (`while true; pause; done` equivalents in C), test harnesses that need a known wakeup point per signal, signal-driven supervisor processes.

This Tier-5 covers the userspace ABI of syscall 34.

### Acceptance Criteria

- [ ] AC-1: `pause()` with a registered SIGUSR1 handler, then `kill(self, SIGUSR1)` from a separate thread: handler runs; `pause` returns `-1` with `errno == EINTR`.
- [ ] AC-2: `pause()` with `SIG_IGN` for all catchable signals, then `kill(self, SIGUSR1)`: pause does NOT return; signal is ignored.
- [ ] AC-3: `pause()` then `kill(self, SIGTERM)` with default disposition: process is terminated; pause does not "return" in the observable sense.
- [ ] AC-4: `pause()` then `kill(self, SIGSTOP)` then `SIGCONT`: pause continues to sleep; only a catchable signal awakens it.
- [ ] AC-5: `pause()` after `sigblock(SIGUSR1)` then `kill(self, SIGUSR1)`: pause does NOT return (signal pending but blocked).
- [ ] AC-6: `pause()` in a pthread cancellation-enabled thread, then `pthread_cancel`: thread is cancelled (`PTHREAD_CANCELED` returned).
- [ ] AC-7: `pause()` does NOT modify the signal mask.
- [ ] AC-8: `pause()` is async-signal-safe (callable from signal handlers).
- [ ] AC-9: Two threads each calling `pause()`: a `kill(pid, SIGUSR1)` (process-directed) awakens exactly one (signal pending on signal->shared).
- [ ] AC-10: A directed `tgkill(pid, tid, SIGUSR1)` to a specific paused thread: awakens that thread only.

### Architecture

Rookery surface in `kernel/signal/syscalls.rs`:

```rust
pub fn sys_pause() -> isize {
    loop {
        Task::set_state(TASK_INTERRUPTIBLE);
        if Signal::pending(current()) {
            break;
        }
        Sched::schedule();
    }
    Task::set_state(TASK_RUNNING);
    /* Signal delivery (handler dispatch) happens on return-to-user; we just report EINTR. */
    -ERESTARTNOHAND as isize
}
```

`-ERESTARTNOHAND` is the kernel-internal sentinel that the signal-delivery layer converts to `-EINTR` on syscall-return to userspace (because `pause(2)` has no `SA_RESTART` semantics — it must not auto-restart, by design).

Wakeup path: a signal that is unblocked + caught (or default-terminate, or default-ignore-with-wakeup) sets `TIF_SIGPENDING` and calls `wake_up_state(task, TASK_INTERRUPTIBLE)`, which puts the task back on the runqueue. The loop then notices `Signal::pending` and breaks.

### Out of Scope

- `kernel/signal.md` Tier-3: signal delivery, pending sets.
- `kernel/sched/core.md` Tier-3: `TASK_INTERRUPTIBLE` semantics and schedule().
- `sigsuspend.md` modern sibling (atomic mask-set + block).
- `ppoll.md` / `pselect.md` siblings with timeout.
- `rt_sigaction.md` (signal disposition).
- glibc / musl userspace wrappers.
- Implementation code.

### signature

```c
int pause(void);
```

Rust ABI shim:

```rust
pub fn sys_pause() -> isize;
```

Syscall number: **34** (x86_64). Not present on arm64 / generic syscall ABI (use `ppoll(2)` with NULL fdset + NULL timeout, or `sigsuspend(2)`).

### parameters

(none)

### return

- **Never returns success.**
- **Always returns `-1`** with `errno == EINTR` once a signal is delivered whose disposition is either "caught by handler" or "terminate process". In the terminate case the process actually exits without the syscall returning to userspace; the `EINTR` return is observed only when a handler was installed.

### errors

| errno | Trigger |
|---|---|
| `EINTR` | Signal delivered with installed handler; handler executed; syscall returned. |

(No other errors are possible: `pause(2)` takes no arguments to validate, takes no resources to allocate, and cannot fault.)

### abi surface

```text
__NR_pause (x86_64) = 34
__NR_pause (i386)   = 29
__NR_pause (arm64 / generic) = unavailable
                              (emulated by glibc via ppoll(NULL, 0, NULL, NULL))

/* Behavior */
- TASK state: TASK_INTERRUPTIBLE
- Schedule: schedule() in a loop
- Termination: signal_pending(current) breaks loop
- Return: always -1 / EINTR
```

### compatibility contract

REQ-1: Syscall number is **34** on x86_64; not present on arm64 / generic. ABI-stable since 1.0 on x86.

REQ-2: No arguments; no validation; no errors possible other than `EINTR`.

REQ-3: Process placed in `TASK_INTERRUPTIBLE` state.

REQ-4: Blocks until `signal_pending(current) != 0`.

REQ-5: On signal:
- If signal disposition is `SIG_IGN`: not awakened (signal_pending may transiently set but loop re-sleeps).
  - Note: glibc may emulate `pause(2)` via `ppoll(NULL, 0, NULL, NULL)` on architectures lacking `pause`; semantics identical.
- If signal disposition is a handler: handler invoked between `schedule()` return and syscall return; syscall returns `-1` / `EINTR`.
- If signal disposition is default-terminate: process terminated without returning.

REQ-6: Stop signals (`SIGSTOP`, `SIGTSTP`) do not cause `pause(2)` to return; the task is stopped and on resume continues to sleep.

REQ-7: `SIGCONT` to a paused task: continues sleeping (no return).

REQ-8: No timeout: `pause(2)` waits indefinitely.

REQ-9: Async-signal-safe per POSIX.1-2008.

REQ-10: Cancellation point: `pause(2)` IS a pthread cancellation point. A pending cancellation request will be acted on.

REQ-11: Interaction with `pselect`/`ppoll`/`sigsuspend`: `pause(2)` is equivalent to `sigsuspend(&current_mask)` only if no signal mask change is needed (and there is no atomic-mask-set requirement); otherwise prefer `sigsuspend(2)` to avoid the well-known race between `sigprocmask` and `pause`.

REQ-12: `pause(2)` does NOT alter the signal mask.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `interruptible_state` | INVARIANT | task state == `TASK_INTERRUPTIBLE` during the inner `schedule()` call. |
| `loop_terminates_on_signal_pending` | INVARIANT | `signal_pending(current)` ⟹ loop breaks within one iteration. |
| `ignored_signal_does_not_break` | INVARIANT | signal with `SIG_IGN` disposition does NOT cause break (signal_pending cleared by signal layer). |
| `blocked_signal_does_not_break` | INVARIANT | signal in `current->blocked` set does NOT cause break. |
| `returns_eintr` | INVARIANT | return path → userspace ⟹ `errno == EINTR`. |
| `no_mask_modification` | INVARIANT | `current->blocked` unchanged across call. |

### Layer 2: TLA+

`uapi/pause.tla`:
- Variables: `task.state ∈ {RUNNING, INTERRUPTIBLE}`, `task.signal_pending`, `task.blocked_mask`, `task.handlers`.
- Properties:
  - `safety_no_mask_change` — `task.blocked_mask` unchanged.
  - `safety_blocked_signal_ignored` — signal in `blocked_mask` does NOT awaken.
  - `safety_ignored_signal_no_return` — signal with `SIG_IGN` does NOT awaken.
  - `liveness_signal_eventually_wakes` — unblocked + non-ignored signal eventually awakens the task.
  - `liveness_no_spurious_wakeup` — wakeup ⟺ signal_pending.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_pause` post: task state == TASK_RUNNING; return == -ERESTARTNOHAND | `sys_pause` |
| `sys_pause` inner loop: signal_pending breaks; no other exit | `sys_pause` |
| `signal_delivery` post-syscall: `pause` ERESTARTNOHAND → user EINTR | signal layer |

### Layer 4: Verus/Creusot functional

Per-`pause(2)` man page; per-LTP `testcases/kernel/syscalls/pause/*`; per-glibc `sysdeps/unix/syscalls.list`; per-POSIX.1-2008.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

pause reinforcement:

- **Per-TASK_INTERRUPTIBLE strict** — defense against per-uninterruptible-pause that would resist `kill -9` (which is delivered as SIGKILL and must always wake).
- **Per-signal-mask immutability strict** — defense against per-mask-leak: `pause` MUST NOT mutate `current->blocked`.
- **Per-ERESTARTNOHAND, NOT ERESTARTSYS** — defense against per-auto-restart: `pause` must NOT be silently restarted, because doing so would make the EINTR semantics unobservable.
- **Per-spurious-wakeup loop guard** — defense against per-spurious-wakeup races; the `loop { ... signal_pending ... }` pattern is robust.

### grsecurity/pax-style reinforcement

- **PaX UDEREF irrelevant** — `pause` takes no arguments; no userspace pointer dereference.
- **PAX_RANDKSTACK on every entry** — randomise kstack on every `pause` entry; defense against per-stack-spray attacks that lie in wait for a signal-driven syscall return.
- **GRKERNSEC_HIDESYM on `current->signal`, `current->blocked`** — signal struct addresses excluded from `/proc/kallsyms` and `/proc/<pid>/status` leak under `kptr_restrict ≥ 2`.
- **CAP_KILL strict in target user-ns** — although `pause` itself doesn't send signals, the wakeup depends on `kill(2)`/`tgkill(2)` which must check CAP_KILL in target's user-ns; defense against per-userns CAP-laundering.
- **Per-uid pause rate-limit** — `pause` storms (rapid pause/wake cycles via SIGUSR1) audit-logged as a possible side-channel; defense against per-signal-driven CPU-time-amplification DoS.
- **Audit per-process pause depth** — task spending > 99% of wallclock in `pause` is fine; task using `pause`+`kill` as a real-time event loop at > 1000 events/s audit-logged.
- **GRKERNSEC_RAND_USERID + signal-disposition pinning** — sandbox profiles pin which signals may awaken pause; defense against per-signal-injection from another sandbox.
- **Refuse pause from setuid programs without explicit signal mask** — under hardened policy, setuid programs calling `pause` must first explicitly set up `sigprocmask`; defense against per-setuid race between mask state and pause entry.
- **Deprecated-syscall ENOSYS gate (off by default)** — grsec hardened policy may force `pause(2)` to `-ENOSYS`, forcing applications onto `sigsuspend(2)` which atomically sets a mask before blocking and avoids the historical pause-vs-sigprocmask race; defense against per-legacy-API surface.
- **Pthread cancellation honoured strict** — defense against pause-as-stuck-thread; cancellation must actually cancel.
- **No timeout means no implicit hang** — pair `pause` with the watchdog timer subsystem; under hardened policy, processes that have been in pause for > N hours can be optionally signalled with a configurable disposition.
- **GRKERNSEC_PROC_USER restrict pause-state observability** — `/proc/<pid>/status:State: S (sleeping)` revealing pause-state may be coarsened to `?` under hardened policy; defense against per-process-state reconnaissance.
- **Coarse timer interaction** — under GRKERNSEC_CLOCK_RESOLUTION pause-wakeup timestamps are not exposed at high resolution; defense against per-wakeup-latency side-channel.

