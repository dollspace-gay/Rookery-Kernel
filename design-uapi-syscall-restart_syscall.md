---
title: "Tier-5 syscall: restart_syscall(2) — syscall 219"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`restart_syscall(2)` is an **internal-only re-entry vehicle**: it is NEVER intended to be invoked directly by userspace and is silently used by the kernel to resume a long-blocking syscall (e.g., `nanosleep`, `clock_nanosleep`, `poll`, `select`, `futex`, `read` on a slow fd) that was interrupted by a signal whose handler returned. When such a syscall blocks, it records a `struct restart_block` in `current->restart_block` containing a function pointer (`fn`) and a small typed payload (`arg0..arg7`). On signal-handler return, the kernel arranges the userspace return address to point at the `restart_syscall` entry; userspace `rt_sigreturn` then re-enters the kernel via syscall 219, which dispatches `current->restart_block.fn` to resume the original operation with the original arguments **adjusted by elapsed time** so deadlines remain absolute. If userspace deliberately issues syscall 219 with no pending restart, the kernel returns `-EINTR` via `do_no_restart_syscall` (the default no-op handler). Critical for: SA_RESTART semantics, transparent deadline preservation across signal handlers, `nanosleep` remaining-time bookkeeping.

This Tier-5 covers the userspace ABI (or rather: the deliberate **non-ABI**) of syscall 219. Per-restart-block dispatch tables and per-syscall restart helpers (`hrtimer_nanosleep_restart`, `do_futex_restart`, etc.) are owned by `kernel/signal.md` and each blocking-syscall's Tier-3 doc.

### Acceptance Criteria

- [ ] AC-1: Direct userspace issue (no pending restart) ⟹ `-EINTR`.
- [ ] AC-2: `nanosleep(2s)` interrupted at 1.2s by SIGALRM with SA_RESTART; handler returns; restart_syscall called automatically; nanosleep completes ~800ms later.
- [ ] AC-3: After completion, second direct issue ⟹ `-EINTR` (fn reset to do_no_restart_syscall).
- [ ] AC-4: `futex(FUTEX_WAIT, timeout)` interrupted: restart resumes with adjusted remaining-time.
- [ ] AC-5: Child of fork issues restart_syscall directly ⟹ `-EINTR` (no inheritance).
- [ ] AC-6: exec resets restart_block; post-exec direct issue ⟹ `-EINTR`.
- [ ] AC-7: `poll(timeout)` interrupted: resume preserves remaining timeout.
- [ ] AC-8: Audit shows the original syscall on resume (e.g., `clock_nanosleep`), not `restart_syscall`.
- [ ] AC-9: Handler that returns `-ERESTART_RESTARTBLOCK` again: loop terminates (kernel re-enters resume; eventually completes or signal-cancels).

### Architecture

Rookery surface in `kernel/signal.rs`:

```rust
pub fn sys_restart_syscall() -> isize {
    let rb = &mut current().restart_block;
    let fn_ptr = rb.fn_;
    /* Defense: reset to no-op BEFORE calling so a re-entrant signal cannot re-fire */
    rb.fn_ = do_no_restart_syscall;
    fn_ptr(rb)
}

pub fn do_no_restart_syscall(_rb: &mut RestartBlock) -> isize {
    -EINTR
}
```

Per-syscall handler example (`hrtimer_nanosleep_restart`):
1. let now = ktime_get_clock(rb.nanosleep.clockid);
2. if now >= rb.nanosleep.expires:
   - return 0;          // already past deadline
3. let new_expires = rb.nanosleep.expires;
4. /* Re-arm hrtimer with absolute expires */
5. let mode = HRTIMER_MODE_ABS;
6. let r = hrtimer_nanosleep_abs(new_expires, mode, rb.nanosleep.clockid);
7. /* On further signal, re-record restart_block (same `expires`) and return -ERESTART_RESTARTBLOCK */
8. r

Per-syscall handler example (`futex_wait_restart`):
1. /* Re-enter futex_wait with original uaddr/val/bitset and remaining timeout */
2. let abs_to = rb.futex.time;
3. let now = ktime_get_clock(rb.futex.clockid);
4. if now >= abs_to: return -ETIMEDOUT;
5. futex_wait(rb.futex.uaddr, rb.futex.val, rb.futex.bitset, abs_to - now, rb.futex.flags)

### Out of Scope

- `clock_nanosleep.md` / `nanosleep.md` / `futex.md` / `poll.md` / `epoll_pwait.md` — the syscalls that install restart_blocks.
- `kernel/signal.md` Tier-3: signal restart flow, `-ERESTART_*` codes.
- `kernel/time/hrtimer.md` Tier-3: hrtimer_nanosleep_restart.
- `kernel/futex.md` Tier-3: futex_wait_restart.
- Architecture-specific syscall entry/exit (arch/x86/entry/common.c).
- glibc / musl signal-restart wrappers.
- Implementation code.

### signature

```c
long restart_syscall(void);
```

Rust ABI shim:

```rust
pub fn sys_restart_syscall() -> isize;
```

Syscall number: **219**.

### parameters

(none) — all state is in `current->restart_block`.

### return

- **Success**: whatever `current->restart_block.fn(restart_block)` returns (the resumed syscall's own return value).
- **No active restart**: `-EINTR` (the default `do_no_restart_syscall` handler).
- Userspace MUST NOT issue this directly; doing so reliably returns `-EINTR`.

### errors

| errno | Trigger |
|---|---|
| `EINTR` | No restart pending; `current->restart_block.fn == do_no_restart_syscall` |
| (any) | Whatever the resumed handler returns (e.g., `-EINTR` again if another signal arrives, `0` on completion, `-EFAULT` on bad remaining-time copy_to_user) |

### abi surface

Userspace MUST NOT issue this syscall manually. The kernel installs `restart_syscall` as the resumption vehicle via:

1. Blocking syscall sets `current->restart_block.fn = my_restart_fn` and populates `arg0..arg7` with state.
2. Signal arrives; handler runs in userspace.
3. Kernel signal-return path sees `regs->ax == -ERESTART_RESTARTBLOCK` and rewrites `regs->ax = __NR_restart_syscall` (219) so the next syscall entry is 219.
4. Userspace `rt_sigreturn` returns to userspace; the syscall trampoline re-enters the kernel via 219.
5. Kernel `sys_restart_syscall` calls `current->restart_block.fn(&current->restart_block)`.
6. After completion, kernel resets `fn = do_no_restart_syscall` so a second invocation cannot accidentally re-resume.

`struct restart_block`:

```c
struct restart_block {
    unsigned long arch_data;
    long (*fn)(struct restart_block *);
    union {
        struct { clockid_t clockid; enum timespec_type type;
                 struct __kernel_timespec __user *rmtp; u64 expires; }
            nanosleep;
        struct { u32 __user *uaddr; u32 val; u32 flags;
                 u32 bitset; u64 time; u32 __user *uaddr2; }
            futex;
        struct { struct __kernel_timespec __user *tsp;
                 long left; long lim; sigset_t __user *sset; }
            poll;
        /* ... */
    };
};
```

Per-syscall restart functions registered:

| Original syscall | `restart_block.fn` |
|---|---|
| `nanosleep` / `clock_nanosleep` | `hrtimer_nanosleep_restart` |
| `futex(FUTEX_WAIT)` | `futex_wait_restart` |
| `poll` / `ppoll` | `do_restart_poll` |
| `epoll_pwait` | `do_restart_epoll_pwait` |
| `select` / `pselect` | `do_compat_pselect6_restart` (or arch-specific) |
| (none active) | `do_no_restart_syscall` |

`do_no_restart_syscall(rb) -> -EINTR;`

### compatibility contract

REQ-1: Userspace-direct issue returns `-EINTR`:
- `current->restart_block.fn == do_no_restart_syscall` (the default after every syscall and at task creation).
- Calling `sys_restart_syscall` invokes that function which returns `-EINTR`.
- This is BY DESIGN: there is no way for an attacker to invoke an arbitrary restart handler without first having blocked in a restart-supporting syscall.

REQ-2: Internal kernel-arranged invocation:
- After a blocking syscall returns `-ERESTART_RESTARTBLOCK`:
  - `setup_rt_frame` records `regs->ax` as `__NR_restart_syscall` (219).
  - On `rt_sigreturn`, userspace resumes at the syscall instruction with rax==219.
- `sys_restart_syscall` dispatches to `current->restart_block.fn`.

REQ-3: Reentrance protection:
- After dispatch, `current->restart_block.fn = do_no_restart_syscall` to prevent accidental double-resume.

REQ-4: Per-handler responsibility:
- The restart handler MUST adjust deadlines: e.g., `hrtimer_nanosleep_restart` computes `expires - now` for the remaining nanosleep duration; finite-deadline syscalls preserve absolute expiry.
- If the deadline has already passed, the handler returns the syscall's "complete" result (e.g., `0` for `nanosleep`, `0` for `futex_wait` timeout).

REQ-5: `restart_block.arch_data`:
- Arch-specific scratch (e.g., x86 syscall args).
- Some architectures use it to record original syscall number.

REQ-6: Signal restarting semantics:
- `SA_RESTART` set on the signal: kernel issues `-ERESTARTSYS` (or `_RESTARTNOINTR` / `_RESTARTNOHAND`); kernel restarts via direct syscall-number replay, NOT via syscall 219.
- `restart_syscall` path is specifically for `-ERESTART_RESTARTBLOCK`: syscalls whose arguments cannot be naively replayed because they include deadlines (`nanosleep`'s remaining time, `poll`'s remaining timeout).

REQ-7: Audit:
- Successful resume audits as the **original** syscall (via `audit_syscall_entry` re-entered).
- Direct userspace issue audits as `restart_syscall` with `-EINTR`.

REQ-8: No copy_from_user:
- `restart_block` is kernel-private; userspace cannot supply arguments.

REQ-9: Per-task scope:
- `restart_block` lives in `task_struct`; not inherited across fork (child gets `do_no_restart_syscall`).
- exec also resets it.

REQ-10: Per-handler errors:
- A buggy handler returning `-ERESTART_RESTARTBLOCK` again would loop; kernel asserts this in `DEBUG_KERNEL` builds.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `direct_userspace_issue_returns_einter` | INVARIANT | restart_block.fn == do_no_restart_syscall ⟹ -EINTR |
| `fn_reset_before_dispatch` | INVARIANT | rb.fn_ overwritten to no-op before calling old fn |
| `restart_block_per_task` | INVARIANT | rb lives in task_struct; not inherited on fork |
| `no_userspace_args` | INVARIANT | syscall reads no userspace memory |

### Layer 2: TLA+

`uapi/restart_syscall.tla`:
- Variables: `rb.fn_`, `expires`, `now`.
- Properties:
  - `safety_no_arbitrary_dispatch` — userspace cannot select rb.fn_; only kernel-internal blocking paths can install it.
  - `safety_idempotent_after_completion` — after first dispatch, second restart_syscall returns -EINTR.
  - `liveness_deadline_resumes_or_completes` — every restart_block eventually terminates (success, timeout, or error).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_restart_syscall` post: rb.fn_ == do_no_restart_syscall | `sys_restart_syscall` |
| `do_no_restart_syscall` post: returns -EINTR | `do_no_restart_syscall` |
| `hrtimer_nanosleep_restart` post: expires preserved across resume | per-handler |
| no copy_from_user / no userspace touch | `sys_restart_syscall` |

### Layer 4: Verus/Creusot functional

Per-`restart_syscall(2)` man page (note: man page explicitly states "should never be called directly by application programs"), `Documentation/admin-guide/restart-syscall.rst`, glibc's NPTL futex/sleep restart wrappers.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

restart_syscall reinforcement:

- **No userspace arguments** — the handler payload comes from kernel state only.
- **fn reset before dispatch** — re-entrant safety; no double-resume.
- **Per-task restart_block** — no cross-task sharing.
- **Fork/exec reset** — `do_no_restart_syscall` default.
- **Default `do_no_restart_syscall` returns `-EINTR`** — direct issue yields a predictable error.

### grsecurity/pax-style reinforcement

- **GRKERNSEC_NO_RESTARTSYSCALL audit-direct-issue** — under hardened policy, any direct userspace invocation of syscall 219 (where `rb.fn_ == do_no_restart_syscall`) emits an audit record `(syscall=219, uid, pid, comm)`. Userspace SHOULD NEVER issue restart_syscall directly; doing so indicates malware probing for kernel re-entry primitives or a libc bug. Audit-direct-issue lets defenders detect both.
- **GRKERNSEC_HIDESYM on `do_no_restart_syscall` and per-handler functions** — restart handler function pointers (`hrtimer_nanosleep_restart`, `futex_wait_restart`, etc.) hidden from `kallsyms` under `kptr_restrict ≥ 2`; defense against ROP-style abuse where attackers would target the indirect call in `sys_restart_syscall`.
- **PaX KERNEXEC on restart-block dispatch** — the indirect call `rb.fn_(rb)` is guarded so that `rb.fn_` MUST point into kernel-`.text` with the executable bit; defense against kernel-write primitive corruption of `rb.fn_` to redirect execution to attacker-controlled memory (CFI-like enforcement for this specific indirect call).
- **`restart_block.fn` validated against whitelist** — under PaX_RAP / hardened-CFI, `rb.fn_` is checked against the small fixed set of registered restart handlers; any mismatch (indicating UAF or kernel-write corruption) triggers a hardened-mode OOPS.
- **PAX_RANDKSTACK on every entry** — random kstack on each call; relevant because restart paths cross-call into hrtimer, futex, and signal subsystems.
- **GRKERNSEC_AUDIT_SYSCALL filtered** — direct restart_syscall issues elevated to high-severity audit even outside the normal syscall-audit policy.
- **Reset `rb.fn_` BEFORE dispatch even on panic-path** — the fn_ reset MUST happen before the handler runs, so even a handler that panics or oopses cannot leave a stale handler installed for the next syscall.
- **Refuse restart if `current.cred` changed since arming** — if the credentials at restart-arming time differ from the credentials at restart-dispatch time (e.g., a setuid syscall ran between), hardened policy returns `-EINTR` instead of dispatching; defense against priv-escalation via a restart that originally ran as one uid but now executes the handler under elevated cred.
- **No-cross-userns restart** — restart_block is task-scoped; even with sharing primitives, hardened policy validates `current.user_ns == rb.userns` at arming time and at dispatch.
- **GRKERNSEC_HIDESYM on `task_struct.restart_block` offset** — offset within task_struct hidden where possible (struct randomization); defense against attackers crafting a kernel-write primitive that targets the fn_ field directly via known offset.
- **Audit on suspicious resume patterns** — if a single task issues more than N restart_syscall dispatches per second, hardened policy logs a "restart storm" alert; defenders detect adversarial signal-injection loops designed to thrash restart paths.
- **`rb.arch_data` zeroed on reset** — defense against information leak where a stale arch-data value could be read by a subsequent restart_block consumer.
- **Per-handler return-value sanity** — handlers must return one of `{0, -ETIMEDOUT, -EINTR, -ERESTART_RESTARTBLOCK, -EFAULT, -EAGAIN}` under hardened policy; any other return triggers a WARN_ONCE and the call returns -EINTR to be conservative.

