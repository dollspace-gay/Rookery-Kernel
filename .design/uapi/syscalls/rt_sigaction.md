# Tier-5 syscall: rt_sigaction(2) — syscall 13

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/signal.c (sys_rt_sigaction, do_sigaction)
  - include/linux/signal.h (struct k_sigaction)
  - include/uapi/asm-generic/signal.h (struct sigaction layout)
  - include/uapi/asm-generic/signal-defs.h (SA_* flags)
  - arch/x86/entry/syscalls/syscall_64.tbl (13  common  rt_sigaction)
-->

## Summary

`rt_sigaction(2)` installs or queries the signal disposition for a given
signal. It is the modern (real-time-signal-aware) replacement for the
historical `sigaction(2)` (which used a smaller `sigset_t`); only
`rt_sigaction` is wired on modern archs. Userspace `sigaction(3)` always
maps to `rt_sigaction(2)` plus a per-arch restorer trampoline.

A disposition consists of:
- **Handler**: `void (*sa_handler)(int)` for classic, or `void
  (*sa_sigaction)(int, siginfo_t *, void *)` for SA_SIGINFO mode; the
  sentinels `SIG_DFL` (kernel default action) and `SIG_IGN` (ignore).
- **Flags**: `SA_NOCLDSTOP`, `SA_NOCLDWAIT`, `SA_SIGINFO`,
  `SA_UNSUPPORTED`, `SA_EXPOSE_TAGBITS`, `SA_ONSTACK`, `SA_RESTART`,
  `SA_NODEFER`, `SA_RESETHAND`, plus arch-specific bits.
- **Mask**: a `sigset_t` of signals to add to the task's signal mask
  while the handler runs (the handled signal is added automatically
  unless `SA_NODEFER`).
- **Restorer**: arch-specific trampoline pointer; libc supplies a small
  PC-relative stub that issues `rt_sigreturn(2)` after the handler
  returns. On archs that define `SA_RESTORER`, the kernel uses this
  pointer as the return address pushed on the signal stack; on others
  (arm64, riscv, loongarch) the vDSO provides a kernel-mapped trampoline.

`SIGKILL` and `SIGSTOP` CANNOT be caught, blocked, or ignored — attempts
to set a disposition for them return `-EINVAL`.

This Tier-5 covers `kernel/signal.c::SYSCALL_DEFINE4(rt_sigaction, ...)`
(~50 lines of entry code) plus its `do_sigaction` workhorse.

## Signature

```c
long rt_sigaction(int signum,
                  const struct sigaction *act,
                  struct sigaction *oact,
                  size_t sigsetsize);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `signum` | `int` | in | Signal number `1..SIGRTMAX (64)`. `SIGKILL (9)` and `SIGSTOP (19)` MUST return `-EINVAL`. |
| `act` | `const struct sigaction *` | in | Pointer to a new disposition. `NULL` means "do not change". |
| `oact` | `struct sigaction *` | out | Pointer to receive the previous disposition. `NULL` means "do not return". |
| `sigsetsize` | `size_t` | in | Size in bytes of the `sa_mask` field — MUST equal `sizeof(sigset_t)` for the kernel's pointer-width. On LP64, this is `8`. |

## Return value

| Value | Meaning |
|---|---|
| `0`  | Success. `act` (if non-NULL) installed; `oact` (if non-NULL) populated. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `signum < 1` or `signum > _NSIG (64)`. |
| `EINVAL` | `signum == SIGKILL (9)` or `signum == SIGSTOP (19)`. |
| `EINVAL` | `sigsetsize != sizeof(sigset_t)` for the kernel's native pointer width. |
| `EFAULT` | `act` non-NULL but not readable, or `oact` non-NULL but not writable. |

## ABI surface

```text
__NR_rt_sigaction (x86_64)    = 13
__NR_rt_sigaction (i386)      = 174
__NR_rt_sigaction (generic)   = 134   /* arm64/riscv/loongarch */
__NR_rt_sigaction (powerpc)   = 173
__NR_rt_sigaction (s390x)     = 174
__NR_rt_sigaction (sparc)     = 101
__NR_rt_sigaction (mips O32)  = 4194
__NR_rt_sigaction (mips N64)  = 5013
```

### `struct sigaction` (asm-generic — `uapi/asm-generic/signal.h:75`)

```c
struct sigaction {
    __sighandler_t sa_handler;
    unsigned long  sa_flags;
#ifdef SA_RESTORER
    __sigrestore_t sa_restorer;
#endif
    sigset_t       sa_mask;        /* mask LAST for extensibility */
};
```

`SA_RESTORER` is defined on: x86, x32, x86_64, sparc, alpha, parisc, mips.
NOT defined on: arm64, riscv, loongarch (the vDSO supplies the
trampoline; the field is absent from `struct sigaction` entirely).

When `SA_SIGINFO` is set, `sa_handler` is interpreted as
`void (*sa_sigaction)(int signo, siginfo_t *si, void *ucontext)`. The
union-like overload is purely a userspace ABI convention; the kernel
stores it as `sa_handler` and calls it with the 3-arg signature when
`SA_SIGINFO` is set.

### `sa_flags` bits (`uapi/asm-generic/signal-defs.h`)

```text
SA_NOCLDSTOP      = 0x00000001    /* SIGCHLD: don't queue for stop/continue */
SA_NOCLDWAIT      = 0x00000002    /* SIGCHLD: don't create zombies */
SA_SIGINFO        = 0x00000004    /* 3-arg handler with siginfo + ucontext */
SA_UNSUPPORTED    = 0x00000400    /* feature-probe bit; reflected back */
SA_EXPOSE_TAGBITS = 0x00000800    /* expose ARM MTE / sparc ADI tag bits in si_addr */
SA_ONSTACK        = 0x08000000    /* deliver on the sigaltstack */
SA_RESTART        = 0x10000000    /* restart slow syscalls (BSD semantics) */
SA_NODEFER        = 0x40000000    /* don't auto-mask the delivered signal */
SA_RESETHAND      = 0x80000000    /* reset to SIG_DFL after first delivery */

SA_NOMASK   = SA_NODEFER        /* historic alias */
SA_ONESHOT  = SA_RESETHAND      /* historic alias */

/* Kernel-internal (never reflected to userspace): */
SA_IMMUTABLE      = 0x00800000   /* set on init's SIG_IGN dispositions in init pid_ns */
```

### Disposition sentinels

```text
SIG_DFL = (__sighandler_t) 0
SIG_IGN = (__sighandler_t) 1
SIG_ERR = (__sighandler_t) -1     /* never installed; only returned by signal(2) */
```

### Restorer trampoline

The libc-supplied restorer is a tiny per-arch stub that the kernel sets
as the return address on the signal stack. After the user handler
returns, control transfers to this stub which executes:

```text
x86_64 restorer:    mov $__NR_rt_sigreturn, %rax; syscall
arm/aarch32:        mov r7, #__NR_rt_sigreturn;   svc 0
arm64 (vDSO):       mov x8, #__NR_rt_sigreturn;   svc 0
i386 restorer:      pop %eax; mov $__NR_rt_sigreturn, %eax; int 0x80
riscv (vDSO):       li a7, __NR_rt_sigreturn;     ecall
```

`rt_sigreturn(2)` restores the pre-signal context from the signal-frame
on the stack and never returns to the restorer.

## Compatibility contract

REQ-1: The syscall number is **13** on x86_64, **134** on generic-syscall
archs. ABI-stable forever.

REQ-2: `sigsetsize` MUST equal the kernel's native `sizeof(sigset_t)`:
- 64-bit kernel: 8 bytes.
- 32-bit kernel: 8 bytes (two longs, see `_NSIG_WORDS`).
- Compat (32-bit userspace on 64-bit kernel): handled by
  `compat_rt_sigaction` with `sizeof(compat_sigset_t)`.

Mismatch MUST return `-EINVAL`.

REQ-3: `signum` MUST be in `[1, _NSIG]` (= `[1, 64]`). Out-of-range MUST
return `-EINVAL`.

REQ-4: `signum == SIGKILL (9)` or `signum == SIGSTOP (19)` MUST return
`-EINVAL`. These dispositions are immutable.

REQ-5: If `act != NULL`:
- Copy `struct sigaction` from user, including `sa_handler`, `sa_flags`,
  `sa_restorer` (if SA_RESTORER arch), `sa_mask`.
- `sa_mask` automatically excludes `SIGKILL` and `SIGSTOP` (kernel
  sanitizes by AND-NOT'ing them).
- `sa_flags & SA_UNSUPPORTED` is preserved verbatim on readback (the
  feature-probe bit lets userspace test which flags the kernel
  supports).
- `sa_flags & ~(known supported bits)` MAY be silently masked OR
  preserved (per-arch implementation detail; mainline preserves them).

REQ-6: If `oact != NULL`:
- Write the PREVIOUS disposition into `*oact` AFTER installing the new
  one (atomic swap semantics).
- If `act == NULL`: just read out the current disposition.

REQ-7: `SA_NOCLDSTOP` (SIGCHLD only): if set on the handler for
`SIGCHLD`, the kernel does NOT generate `SIGCHLD` for child stop/cont
events; only for exit.

REQ-8: `SA_NOCLDWAIT` (SIGCHLD only): if set on the handler for
`SIGCHLD`, exiting children are auto-reaped (no zombies). Compatible
with `SIG_DFL` and `SIG_IGN`.

REQ-9: `SA_SIGINFO`: kernel marshals `siginfo_t *` + `ucontext_t *` into
the signal-stack arguments. The handler is called as a 3-arg function.

REQ-10: `SA_ONSTACK`: handler delivered on `sigaltstack` if one is
configured and not currently in use; otherwise on current stack.

REQ-11: `SA_RESTART`: slow syscalls (read/write on slow devices, wait,
accept, etc.) interrupted by THIS signal are restarted via
`-ERESTARTSYS` semantics; without `SA_RESTART`, they return `-EINTR`.

REQ-12: `SA_NODEFER` (`SA_NOMASK`): the signal being delivered is NOT
added to the task's mask while the handler runs — handler can be
re-entered.

REQ-13: `SA_RESETHAND` (`SA_ONESHOT`): after first delivery, disposition
is reset to `SIG_DFL`. Combines reasonably with `SA_NODEFER` for
"deliver-once, no auto-mask".

REQ-14: `SA_EXPOSE_TAGBITS` (arm64 MTE, sparc ADI): when set, the
`si_addr` field of the delivered `siginfo` retains the address-tag bits
of the faulting address; without it, the kernel zeros the top byte.

REQ-15: Restorer trampoline:
- On arches with `SA_RESTORER`: `act->sa_restorer` MUST be a non-NULL
  pointer to a trampoline; the kernel uses it as the return address
  pushed on the signal stack.
- On arches without `SA_RESTORER`: the kernel ignores the field and
  uses the vDSO-provided trampoline.

REQ-16: Atomicity:
- `do_sigaction` holds `sighand->siglock`.
- The installation is ordered: first read current (for oact), then
  write new — all under one lock.
- Concurrent signal delivery on another CPU either sees the OLD or NEW
  disposition consistently — never a torn mid-write state.

REQ-17: After install, if the disposition transitions to `SIG_IGN` or
`SIG_DFL` (for a signal whose default is "ignore", e.g. SIGCHLD,
SIGURG, SIGWINCH), any currently-queued instances of that signal in the
task's pending queue are flushed.

REQ-18: SA_IMMUTABLE (kernel-internal): set on init (pid 1) for fatal
signals when running as pid_ns init; userspace cannot remove the
disposition. Read-side: kernel returns `-EINVAL` on
`rt_sigaction(...)` for an immutable disposition.

REQ-19: Audit: AUDIT_SYSCALL record on every `rt_sigaction` includes
`signum`, the new handler value, and a snapshot of `sa_flags`.

REQ-20: Multithreaded process: disposition is per-`sighand_struct`,
which is shared across all threads in the same group (unless
`CLONE_SIGHAND` is 0, which only `fork` does). So `rt_sigaction` in one
thread affects all threads in the same process.

## Acceptance Criteria

- [ ] AC-1: Syscall number is 13 on x86_64; 134 on generic.
- [ ] AC-2: `rt_sigaction(SIGUSR1, NULL, &old, 8)`: returns 0; old contains current SIGUSR1 disposition.
- [ ] AC-3: `rt_sigaction(SIGKILL, &act, NULL, 8)` → `-EINVAL`.
- [ ] AC-4: `rt_sigaction(SIGSTOP, &act, NULL, 8)` → `-EINVAL`.
- [ ] AC-5: `rt_sigaction(0, ...)` → `-EINVAL`.
- [ ] AC-6: `rt_sigaction(65, ...)` → `-EINVAL`.
- [ ] AC-7: `rt_sigaction(SIGUSR1, &act, NULL, 4)` (sigsetsize wrong) → `-EINVAL`.
- [ ] AC-8: `rt_sigaction(SIGUSR1, bad_ptr, NULL, 8)` → `-EFAULT`.
- [ ] AC-9: `rt_sigaction(SIGUSR1, &act, &old, 8)`: `*old` reflects pre-call disposition; `act` becomes current.
- [ ] AC-10: `SA_RESETHAND`: after handler runs once, disposition is `SIG_DFL`.
- [ ] AC-11: `SA_NODEFER`: handler can recurse on same signal (mask not auto-modified).
- [ ] AC-12: `SA_SIGINFO`: handler receives `siginfo_t *` second arg.
- [ ] AC-13: `SA_RESTART`: `read()` on a pipe interrupted by SIGUSR1 returns to read; without, returns `-EINTR`.
- [ ] AC-14: `SA_ONSTACK` + `sigaltstack()`: handler stack is at altstack base.
- [ ] AC-15: `SA_NOCLDWAIT` on SIGCHLD: child exits auto-reaped, no zombie.
- [ ] AC-16: `SA_NOCLDSTOP` on SIGCHLD: child stop generates no signal.
- [ ] AC-17: `act->sa_mask` includes `SIGKILL`: kernel silently masks it out; effective mask excludes SIGKILL.
- [ ] AC-18: x86_64 restorer: `act->sa_restorer` MUST be set when `SA_RESTORER` is in `sa_flags` (or libc has supplied it); kernel uses it.
- [ ] AC-19: arm64: `act->sa_restorer` field absent; vDSO trampoline used.
- [ ] AC-20: Multithreaded: rt_sigaction in thread A is observed by thread B (shared sighand).
- [ ] AC-21: Transition to SIG_IGN: previously-queued instances flushed.
- [ ] AC-22: Auditd records `signum`, handler ptr, flags.

## Architecture

```rust
#[syscall(nr = 13, abi = "sysv")]
pub fn sys_rt_sigaction(
    sig: i32,
    act: UserPtr<UapiSigaction>,
    oact: UserPtr<UapiSigaction>,
    sigsetsize: usize,
) -> isize {
    // REQ-2: sigsetsize
    if sigsetsize != size_of::<Sigset>() { return -EINVAL; }
    // REQ-3/4: signum range + immutables
    if sig < 1 || sig > _NSIG as i32 { return -EINVAL; }
    if sig == SIGKILL || sig == SIGSTOP { return -EINVAL; }

    // Copy `act` from user if provided
    let new_ka: Option<KSigaction> = if !act.is_null() {
        Some(KSigaction::from_user(act)?)
    } else { None };

    let mut old_ka = KSigaction::default();
    Signal::do_sigaction(sig, new_ka.as_ref(), Some(&mut old_ka))?;

    if !oact.is_null() {
        old_ka.to_user(oact)?;
    }
    0
}
```

`Signal::do_sigaction(sig, act, oact) -> Result<()>`:
1. let task    = Task::current();
2. let sighand = task.sighand;
3. let _g      = SpinLockGuard::lock(&sighand.siglock);
4. let slot    = &mut sighand.action[(sig - 1) as usize];
5. /* Check immutable */
6. if slot.flags & SA_IMMUTABLE != 0 { return Err(-EINVAL); }
7. /* Copy current to oact */
8. if let Some(o) = oact { *o = slot.clone(); }
9. /* Install new */
10. if let Some(a) = act {
11.   *slot = a.clone();
12.   /* Sanitize: SIGKILL / SIGSTOP cannot be in mask */
13.   sigdelset(&mut slot.mask, SIGKILL);
14.   sigdelset(&mut slot.mask, SIGSTOP);
15.   /* If new disposition ignores/defaults a signal, flush queued */
16.   if Signal::is_ignored(slot, sig) {
17.     Signal::flush_queued_for(task, sig);
18.   }
19. }
20. drop(_g);
21. Ok(())

`KSigaction::from_user(p)`:
1. Read `__sighandler_t sa_handler` (8 bytes).
2. Read `unsigned long sa_flags` (8 bytes on LP64).
3. If arch has SA_RESTORER: read `sa_restorer` (8 bytes).
4. Read `sigset_t sa_mask` (8 bytes on LP64).
5. Total: 24 (no restorer) / 32 (with restorer) bytes.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `signum_range` | TOTAL | sig ∉ [1,_NSIG] ⟹ -EINVAL. |
| `sigkill_sigstop_immutable` | INVARIANT | sig ∈ {SIGKILL, SIGSTOP} ⟹ -EINVAL pre-copy. |
| `sigsetsize_strict` | INVARIANT | sigsetsize ≠ sizeof(sigset_t) ⟹ -EINVAL. |
| `swap_atomic` | INVARIANT | oact reads the disposition that was present BEFORE this call wrote act. |
| `mask_sanitized` | INVARIANT | Post-install: slot.mask & {SIGKILL,SIGSTOP} == 0. |
| `siglock_held_during_swap` | INVARIANT | Slot mutation under sighand.siglock. |

### Layer 2: TLA+

`kernel/rt_sigaction.tla`:
- States: VALIDATE → LOCK → READ_OLD → INSTALL_NEW → MAYBE_FLUSH → UNLOCK → COPY_OUT.
- Concurrent: signal-delivery on another CPU.
- Properties:
  - `safety_no_torn_read` — concurrent get_signal observes either pre- or post-install disposition, not torn.
  - `safety_kill_stop_uncatchable` — slot[SIGKILL] / slot[SIGSTOP] never deviate from SIG_DFL.
  - `liveness_eventual_install` — every successful call installs exactly once.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_rt_sigaction` post: returns 0 ⟹ slot updated atomically; oact has prior value | `sys_rt_sigaction` |
| `from_user` / `to_user` byte-layout matches asm-generic | `KSigaction::from_user`, `to_user` |
| Flush-on-ignore: SIG_IGN ⟹ pending queue cleared for sig | `Signal::flush_queued_for` |

### Layer 4: Verus / Creusot functional

POSIX `sigaction(2)` semantic equivalence (Linux `rt_sigaction` is the
implementation). LTP `rt_sigaction01..04`, `kselftests/sigaction/*` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`rt_sigaction(2)` reinforcement:

- **Per-SIGKILL/SIGSTOP immutable** — defense against per-signal-disable bypass.
- **Per-sigsetsize strict** — defense against per-mask-truncation.
- **Per-siglock-held swap** — defense against per-torn disposition.
- **Per-mask sanitize (clear SIGKILL/SIGSTOP)** — defense against per-mask-hide-kill.
- **Per-flush-queued on ignore transition** — defense against per-stale-signal-after-ignore.
- **Per-SA_IMMUTABLE on init pid 1** — defense against per-init self-disable (cannot SIG_IGN fatal signals on init).
- **Per-SA_RESTORER NULL ⟹ -EFAULT** — defense against per-NULL-trampoline crash.
- **Per-audit record** — defense against per-disposition-change elision.
- **Per-handler pointer validated at delivery** — secondary check that the handler is reachable (no_new_privs interaction; not implemented here but consistent with mainline).

## Grsecurity / PaX surface

- **PAX_RANDKSTACK at syscall entry** — per-rt_sigaction call randomizes
  the kernel stack offset; combined with the brief siglock window, this
  defeats per-syscall stack-layout reuse for ROP.
- **PaX UDEREF on user buffers** — `act`, `oact` accessed via
  UDEREF-toggled / PAN-toggled `get_user`/`put_user`; kernel bug cannot
  silently dereference into user space.
- **PaX MPROTECT** — handler entry pointer must lie in an executable
  page; a non-executable handler is caught at delivery (kernel-side
  MPROTECT enforces W^X for the signal-stack restoring).
- **PaX SEGMEXEC / RANDMMAP** — sa_restorer address is randomized per
  libc load; tracking handler address across exec is defeated.
- **GRKERNSEC_HARDEN_PTRACE** — tracer's rt_sigaction via ptrace
  (`PTRACE_GETSIGINFO`, `PTRACE_SETSIGINFO`) honors yama+grsec; cross-cred
  ptrace cannot install dispositions in tracee.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/status` `SigCgt`,
  `SigBlk`, `SigIgn` masks are subject to per-uid hide-filter.
- **GRKERNSEC_CHROOT** — rt_sigaction permitted in chroot; the SA_NOCLDWAIT
  / SA_NOCLDSTOP child-reap semantics inherit chroot lockdown via
  `chroot_findtask`.
- **GRKERNSEC_BRUTE** — repeated `rt_sigaction(SIGSEGV, SIG_IGN, ...)`
  patterns (ROP probing trying to mask faults) are counted toward
  per-uid brute-force counter.
- **GRKERNSEC_SIGNALS spoof prevention** — at signal-delivery time, the
  kernel ensures the `siginfo.si_code` is kernel-credentialed:
  asynchronous-kill `si_code = SI_USER` carries `si_pid` / `si_uid` of
  the actual sender (real_cred); attacker-supplied `siginfo` via
  `pidfd_send_signal(2)` is gated by the per-grsec `signal_logging` and
  permitted-signo policy.
- **Per-grsec `chroot_caps`** — caller in chroot has bounded capset; this
  does not gate `rt_sigaction` (no priv required) but does gate the
  follow-on `pidfd_send_signal`.
- **Per-grsec `gradm` learning** — RBAC may restrict the set of signals a
  subject is allowed to install handlers for (e.g. forbid SIG_IGN on
  SIGTRAP for non-debugger subjects).

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `rt_sigreturn(2)` (separate Tier-5 doc).
- `rt_sigprocmask(2)` / `rt_sigpending(2)` / `rt_sigsuspend(2)` (separate Tier-5 docs).
- `sigaltstack(2)` (separate Tier-5 doc).
- Signal delivery path (`kernel/signal.c::get_signal`, `do_signal` — Tier-3).
- Compat (32-bit on 64-bit) variant `compat_rt_sigaction` (Tier-3).
- vDSO restorer trampoline layout (Tier-3 arch/<arch>/kernel/vdso/).
- Implementation code.
