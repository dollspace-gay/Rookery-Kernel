# Tier-5 UAPI: include/uapi/asm-generic/signal.h — Signal ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/asm-generic/signal.h (~93 lines)
  - include/uapi/asm-generic/signal-defs.h (~93 lines)
  - include/uapi/asm-generic/siginfo.h (~320 lines, struct siginfo + si_code arms)
  - include/uapi/linux/signal.h (tiny — wraps asm/signal.h)
-->

## Summary

The signal UAPI is the syscall-edge contract for every signal-related syscall — `kill(2)`, `tkill(2)`, `tgkill(2)`, `sigaction(2)` / `rt_sigaction(2)`, `sigprocmask(2)` / `rt_sigprocmask(2)`, `sigaltstack(2)`, `sigsuspend(2)` / `rt_sigsuspend(2)`, `sigreturn(2)` / `rt_sigreturn(2)`, `sigtimedwait(2)` / `rt_sigtimedwait(2)`, `sigqueue(2)` / `rt_sigqueueinfo(2)`, `pidfd_send_signal(2)`, `signalfd4(2)`, and the synchronous arch fault paths that deliver `SIGSEGV`, `SIGBUS`, `SIGILL`, `SIGFPE`, `SIGTRAP`, `SIGSYS`. It defines:

- **Signal numbers** 1..31 standard + 32..64 real-time (`SIGRTMIN`..`SIGRTMAX`).
- **`_NSIG = 64`** — total signal-number space; `_NSIG_BPW = BITS_PER_LONG`; `_NSIG_WORDS = _NSIG/_NSIG_BPW` (1 on 64-bit, 2 on 32-bit).
- **`sigset_t`** — 1024-bit fixed-size mask on the kernel boundary (actually `_NSIG`-bit, but exposed as `unsigned long sig[_NSIG_WORDS]`; glibc enlarges this to 1024 bits as a userspace-side ABI buffer).
- **`old_sigset_t`** — `unsigned long`, the obsolete 32-signal mask for the original `sigprocmask`.
- **`struct sigaction`** — handler + flags + restorer + mask.
- **`struct sigaltstack` / `stack_t`** — alternate-stack descriptor (`ss_sp`, `ss_flags`, `ss_size`).
- **`siginfo_t`** + per-signal `si_code` arms (sender identity, faulting address, child status).
- **`struct sigevent`** + `SIGEV_*` notification methods.
- **Signal-source codes** `SI_USER`, `SI_KERNEL`, `SI_QUEUE`, `SI_TIMER`, `SI_MESGQ`, `SI_ASYNCIO`, `SI_SIGIO`, `SI_TKILL`.
- **`sa_flags` bits** `SA_NOCLDSTOP`, `SA_NOCLDWAIT`, `SA_SIGINFO`, `SA_UNSUPPORTED`, `SA_EXPOSE_TAGBITS`, `SA_ONSTACK`, `SA_RESTART`, `SA_NODEFER` (= `SA_NOMASK`), `SA_RESETHAND` (= `SA_ONESHOT`).
- **`sigprocmask(2)` "how" values** `SIG_BLOCK`, `SIG_UNBLOCK`, `SIG_SETMASK`.
- **Disposition sentinels** `SIG_DFL`, `SIG_IGN`, `SIG_ERR`.
- **Stack-size constants** `MINSIGSTKSZ`, `SIGSTKSZ`.

Critical for: every userspace runtime (libc setjmp/longjmp, pthread cancellation, JVM crash handlers, Go runtime's preemption signals, Rust's `signal-hook`), every debugger (`ptrace` siginfo), every container-init that has to forward SIGTERM to children, every real-time application using `SIGRT*`.

This Tier-5 covers `include/uapi/asm-generic/signal.h` and the signal-defs / siginfo files it pulls in. `include/uapi/linux/signal.h` is tiny (a thin wrapper) — its ABI surface is fully captured here.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `sigset_t` | per-signal mask | `Sigset` |
| `old_sigset_t` | per-legacy 32-sig mask | `OldSigset` (`u64` on 64-bit, `u32` on 32-bit) |
| `__sighandler_t` | per-handler fn ptr | `SignalHandler` |
| `__sigrestore_t` | per-arch restorer trampoline | `SigRestorer` |
| `struct sigaction` | per-disposition descriptor | `Sigaction` |
| `struct sigaltstack` (`stack_t`) | per-alt-stack descriptor | `Sigaltstack` (`StackT`) |
| `siginfo_t` | per-delivery payload | `Siginfo` |
| `struct sigevent` | per-async-notification descriptor | `Sigevent` |
| `__signalfn_t` | per-`void(int)` type | `SignalFn` |
| `__restorefn_t` | per-`void(void)` type | `RestoreFn` |
| `SIG_DFL` / `SIG_IGN` / `SIG_ERR` | per-sentinel handler | shared |
| `SIG_BLOCK` / `SIG_UNBLOCK` / `SIG_SETMASK` | per-mask op | shared |
| `_NSIG` / `_NSIG_BPW` / `_NSIG_WORDS` | per-set sizing | shared |
| `SI_USER`..`SI_TKILL` | per-source code | shared |
| `SIGEV_SIGNAL` / `_NONE` / `_THREAD` / `_THREAD_ID` | per-notify method | shared |

## ABI surface (constants + structs)

### Signal numbers (`uapi/asm-generic/signal.h:11-53`)

```text
SIGHUP    =  1   /* hangup (controlling tty close) */
SIGINT    =  2   /* terminal interrupt (^C) */
SIGQUIT   =  3   /* terminal quit (^\) + core */
SIGILL    =  4   /* illegal instruction + core */
SIGTRAP   =  5   /* debugger / breakpoint + core */
SIGABRT   =  6   /* abort(3) + core */
SIGIOT    =  6   /* alias of SIGABRT (PDP-11 IOT) */
SIGBUS    =  7   /* bus error / misaligned access + core */
SIGFPE    =  8   /* floating-point exception + core */
SIGKILL   =  9   /* uncatchable termination */
SIGUSR1   = 10   /* user-defined #1 */
SIGSEGV   = 11   /* invalid memory reference + core */
SIGUSR2   = 12   /* user-defined #2 */
SIGPIPE   = 13   /* write to broken pipe */
SIGALRM   = 14   /* timer expiration (alarm) */
SIGTERM   = 15   /* polite termination */
SIGSTKFLT = 16   /* stack fault on coprocessor (unused) */
SIGCHLD   = 17   /* child stopped/terminated */
SIGCONT   = 18   /* continue if stopped */
SIGSTOP   = 19   /* uncatchable stop */
SIGTSTP   = 20   /* terminal stop (^Z) */
SIGTTIN   = 21   /* background process read from tty */
SIGTTOU   = 22   /* background process write to tty */
SIGURG    = 23   /* urgent socket condition (OOB) */
SIGXCPU   = 24   /* CPU time limit exceeded + core */
SIGXFSZ   = 25   /* file size limit exceeded + core */
SIGVTALRM = 26   /* virtual timer expired */
SIGPROF   = 27   /* profiling timer expired */
SIGWINCH  = 28   /* window resize */
SIGIO     = 29   /* I/O now possible (async) */
SIGPOLL   = SIGIO /* SysV alias */
SIGPWR    = 30   /* power failure / UPS */
SIGSYS    = 31   /* invalid syscall + core */
SIGUNUSED = 31   /* alias of SIGSYS */

SIGRTMIN  = 32
SIGRTMAX  = _NSIG = 64    /* 32 RT signals SIGRTMIN..SIGRTMIN+31 */
```

`SIGKILL (9)` and `SIGSTOP (19)` cannot be caught, blocked, or ignored. `SIGRTMIN..SIGRTMAX` deliver in priority-order (lower-numbered first) and queue (unlike standard signals which coalesce).

### Sigset sizing (`uapi/asm-generic/signal.h:7-9`)

```text
_NSIG       = 64
_NSIG_BPW   = __BITS_PER_LONG          /* 64 on LP64, 32 on ILP32 */
_NSIG_WORDS = _NSIG / _NSIG_BPW        /* 1 on LP64, 2 on ILP32 */
```

`sigset_t`:

```text
typedef struct {
    unsigned long sig[_NSIG_WORDS];    /* bit i set ⟺ signal (i+1) is in the set */
} sigset_t;

typedef unsigned long old_sigset_t;    /* historical 32-bit mask */
```

Bit-indexing convention: `sigset.sig[(signo-1)/_NSIG_BPW] & (1UL << ((signo-1) % _NSIG_BPW))`.

### Stack-size constants (`uapi/asm-generic/signal.h:55-58`)

```text
MINSIGSTKSZ = 2048    /* generic minimum; archs may override (e.g., arm64 = 5120) */
SIGSTKSZ    = 8192    /* recommended */
```

### `struct sigaction` (`uapi/asm-generic/signal.h:75-83`)

```text
struct sigaction {
    __sighandler_t sa_handler;
    unsigned long  sa_flags;
#ifdef SA_RESTORER
    __sigrestore_t sa_restorer;
#endif
    sigset_t       sa_mask;        /* mask LAST for extensibility */
};
```

Some arches (x86, x32, sparc, alpha, parisc, mips) define `SA_RESTORER` and include the restorer pointer; others (arm64, riscv, loongarch) do not (libc supplies the trampoline via `sa_handler` indirection or directly).

### `sa_flags` bits (`uapi/asm-generic/signal-defs.h:29-69`)

```text
SA_NOCLDSTOP     = 0x00000001   /* SIGCHLD: don't deliver for stop/cont */
SA_NOCLDWAIT     = 0x00000002   /* SIGCHLD: don't create zombies */
SA_SIGINFO       = 0x00000004   /* sa_handler is sa_sigaction (3-arg) */
SA_UNSUPPORTED   = 0x00000400   /* feature-probe bit; reflected back */
SA_EXPOSE_TAGBITS= 0x00000800   /* expose ARM MTE / sparc ADI tag bits in si_addr */
SA_ONSTACK       = 0x08000000   /* use sigaltstack */
SA_RESTART       = 0x10000000   /* restart syscalls (BSD semantics) */
SA_NODEFER       = 0x40000000   /* don't mask the delivered signal */
SA_RESETHAND     = 0x80000000   /* SIG_DFL after first delivery */

SA_NOMASK   = SA_NODEFER        /* historic alias */
SA_ONESHOT  = SA_RESETHAND      /* historic alias */

/* arch-reserved bits (some used internally): */
/* 0x00000008  alpha, mips, parisc */
/* 0x00000010  alpha, parisc */
/* 0x00000020  alpha, parisc, sparc */
/* 0x00000040  alpha, parisc */
/* 0x00000080  parisc */
/* 0x00000100  sparc */
/* 0x00000200  sparc */
/* 0x00010000  mips */
/* 0x00800000  reserved for kernel-internal SA_IMMUTABLE */
/* 0x01000000  x86 */
/* 0x02000000  x86 */
/* 0x04000000  obsolete SA_RESTORER for new arches (do not reuse) */
```

### Disposition sentinels (`uapi/asm-generic/signal-defs.h:88-90`)

```text
SIG_DFL = (__sighandler_t)  0    /* default disposition */
SIG_IGN = (__sighandler_t)  1    /* ignore */
SIG_ERR = (__sighandler_t) -1    /* error from signal(2) */
```

### `sigprocmask(2)` "how" values (`uapi/asm-generic/signal-defs.h:71-79`)

```text
SIG_BLOCK    = 0     /* OR mask into current */
SIG_UNBLOCK  = 1     /* AND ~mask into current */
SIG_SETMASK  = 2     /* replace current */
```

### `struct sigaltstack` / `stack_t` (`uapi/asm-generic/signal.h:85-89`)

```text
typedef struct sigaltstack {
    void __user    *ss_sp;
    int             ss_flags;        /* SS_ONSTACK, SS_DISABLE, SS_AUTODISARM, SS_FLAG_BITS */
    __kernel_size_t ss_size;
} stack_t;
```

`ss_flags`:

```text
SS_ONSTACK      = 1
SS_DISABLE      = 2
SS_AUTODISARM   = (1U << 31)     /* clear ON stack-overflow recovery */
SS_FLAG_BITS    = SS_AUTODISARM  /* mask of all valid flag bits */
```

(defined in `<linux/signal.h>`/`<bits/sigstack.h>`, not in asm-generic/signal.h — captured here as ABI-locked partner.)

### `siginfo_t` source codes (`uapi/asm-generic/siginfo.h:174-181`)

```text
SI_USER      =  0    /* kill(2), sigsend(2), raise(3) */
SI_KERNEL    =  0x80 /* generic kernel-originated */
SI_QUEUE     = -1    /* sigqueue(2) */
SI_TIMER     = -2    /* timer_create(2) expiration */
SI_MESGQ     = -3    /* mq_notify(3) */
SI_ASYNCIO   = -4    /* AIO completion */
SI_SIGIO     = -5    /* queued SIGIO */
SI_TKILL     = -6    /* tkill(2) / tgkill(2) */

#define SI_FROMUSER(siptr)   ((siptr)->si_code <= 0)
#define SI_FROMKERNEL(siptr) ((siptr)->si_code > 0)
```

### Per-signal `si_code` arms (`uapi/asm-generic/siginfo.h:189-310`)

```text
SIGILL si_codes:
  ILL_ILLOPC = 1   /* illegal opcode */
  ILL_ILLOPN = 2   /* illegal operand */
  ILL_ILLADR = 3   /* illegal addressing mode */
  ILL_ILLTRP = 4   /* illegal trap */
  ILL_PRVOPC = 5   /* privileged opcode */
  ILL_PRVREG = 6   /* privileged register */
  ILL_COPROC = 7   /* coprocessor error */
  ILL_BADSTK = 8   /* internal stack error */
  ILL_BADIADDR = 9 /* unimplemented instruction */

SIGFPE si_codes:
  FPE_INTDIV = 1   /* integer divide by zero */
  FPE_INTOVF = 2   /* integer overflow */
  FPE_FLTDIV = 3   /* fp divide by zero */
  FPE_FLTOVF = 4   /* fp overflow */
  FPE_FLTUND = 5   /* fp underflow */
  FPE_FLTRES = 6   /* fp inexact */
  FPE_FLTINV = 7   /* fp invalid op */
  FPE_FLTSUB = 8   /* subscript out of range */
  FPE_FLTUNK = 14  /* undiagnosed fp */
  FPE_CONDTRAP = 15

SIGSEGV si_codes:
  SEGV_MAPERR  = 1   /* address not mapped */
  SEGV_ACCERR  = 2   /* permission denied */
  SEGV_BNDERR  = 3   /* MPX bound violation */
  SEGV_PKUERR  = 4   /* protection-key violation */
  SEGV_ACCADI  = 5   /* sparc ADI mismatch */
  SEGV_ADIDERR = 6   /* sparc ADI disrupting error */
  SEGV_ADIPERR = 7   /* sparc ADI precise error */
  SEGV_MTEAERR = 8   /* arm64 MTE async */
  SEGV_MTESERR = 9   /* arm64 MTE sync */
  SEGV_CPERR   = 10  /* control-protection (shadow stack) */

SIGBUS si_codes:
  BUS_ADRALN     = 1   /* invalid alignment */
  BUS_ADRERR     = 2   /* non-existent address */
  BUS_OBJERR     = 3   /* object-specific HW error */
  BUS_MCEERR_AR  = 4   /* hw memory error, action required */
  BUS_MCEERR_AO  = 5   /* hw memory error, action optional */

SIGTRAP si_codes:
  TRAP_BRKPT    = 1   /* breakpoint */
  TRAP_TRACE    = 2   /* trace step */
  TRAP_BRANCH   = 3   /* branch trace */
  TRAP_HWBKPT   = 4   /* HW breakpoint/watchpoint */
  TRAP_UNK      = 5   /* undiagnosed */
  TRAP_PERF     = 6   /* perf event */
  /* + ptrace event arms PTRACE_EVENT_FORK/.../SECCOMP encoded high */

SIGCHLD si_codes:
  CLD_EXITED    = 1   /* normal exit */
  CLD_KILLED    = 2   /* killed by signal */
  CLD_DUMPED    = 3   /* killed and dumped */
  CLD_TRAPPED   = 4   /* traced child has trapped */
  CLD_STOPPED   = 5   /* child stopped */
  CLD_CONTINUED = 6   /* stopped child continued */

SIGPOLL / generic si_codes:
  POLL_IN       = 1   /* data ready */
  POLL_OUT      = 2   /* output buffer free */
  POLL_MSG      = 3   /* input message */
  POLL_ERR      = 4   /* I/O error */
  POLL_PRI      = 5   /* high-priority input */
  POLL_HUP      = 6   /* device disconnected */

SIGSYS si_codes:
  SYS_SECCOMP   = 1   /* seccomp triggered */
  SYS_USER_DISPATCH = 2 /* syscall user dispatch */

SIGEMT si_codes (arches with SIGEMT):
  EMT_TAGOVF    = 1   /* tag overflow (sparc) */
```

### `siginfo_t` layout (`uapi/asm-generic/siginfo.h:134-139`)

```text
typedef struct siginfo {
    int si_signo;
    int si_errno;
    int si_code;
    /* + per-signal union of payload structures */
} __ARCH_SI_ATTRIBUTES siginfo_t;
```

Total size: 128 bytes (`__SI_MAX_SIZE`) on LP64; some arches use `__ARCH_SI_PREAMBLE_SIZE` to shift padding (mips orders preamble differently).

### `struct sigevent` (`uapi/asm-generic/siginfo.h:312-340`)

```text
SIGEV_SIGNAL    = 0    /* notify via signal */
SIGEV_NONE      = 1    /* no notification (poll) */
SIGEV_THREAD    = 2    /* deliver via thread creation (libc-side) */
SIGEV_THREAD_ID = 4    /* deliver to specific thread (Linux) */

#define SIGEV_MAX_SIZE   64
#define __ARCH_SIGEV_PREAMBLE_SIZE (sizeof(int) * 2 + sizeof(sigval_t))

struct sigevent {
    sigval_t       sigev_value;        /* union { int sival_int; void *sival_ptr; } */
    int            sigev_signo;
    int            sigev_notify;       /* SIGEV_* */
    union {
        int        _pad[(SIGEV_MAX_SIZE - __ARCH_SIGEV_PREAMBLE_SIZE) / sizeof(int)];
        int        _tid;               /* SIGEV_THREAD_ID */
        struct {
            void  (*_function)(sigval_t);
            void   *_attribute;        /* pthread_attr_t * */
        } _sigev_thread;
    } _sigev_un;
};
```

## Compatibility contract

REQ-1: `_NSIG == 64`. `SIGRTMIN == 32`, `SIGRTMAX == _NSIG == 64`. `_NSIG_BPW == BITS_PER_LONG`. `_NSIG_WORDS == 1` on LP64, `2` on ILP32.

REQ-2: Standard signal numbers MUST match the table (`SIGHUP=1 .. SIGSYS=31`). `SIGIOT == SIGABRT == 6`. `SIGPOLL == SIGIO == 29`. `SIGUNUSED == SIGSYS == 31`.

REQ-3: `SIGKILL == 9` and `SIGSTOP == 19` MUST be uncatchable: `sigaction(SIGKILL, ...)` / `sigaction(SIGSTOP, ...)` MUST return `-EINVAL`; `sigprocmask` MUST silently drop them from any block-mask.

REQ-4: `sigset_t` MUST be exactly `_NSIG_WORDS` words of `unsigned long` — 8 bytes on LP64, 8 bytes (`2 * u32`) on ILP32. Bit-indexing: bit `(signo-1)` = signal `signo`.

REQ-5: `struct sigaction` field order MUST match Linux: `sa_handler`, `sa_flags`, optional `sa_restorer`, `sa_mask` LAST. The trailing `sa_mask` placement is load-bearing for future extensibility.

REQ-6: On x86/x86_64/sparc/parisc/mips/alpha/x32, `SA_RESTORER == 0x04000000` is defined and `sa_restorer` is present. On arm64/riscv/loongarch/powerpc, `SA_RESTORER` is NOT defined and the field is omitted.

REQ-7: `sa_flags` values MUST match: `SA_NOCLDSTOP=0x1`, `SA_NOCLDWAIT=0x2`, `SA_SIGINFO=0x4`, `SA_UNSUPPORTED=0x400`, `SA_EXPOSE_TAGBITS=0x800`, `SA_ONSTACK=0x08000000`, `SA_RESTART=0x10000000`, `SA_NODEFER=0x40000000`, `SA_RESETHAND=0x80000000`. Aliases: `SA_NOMASK=SA_NODEFER`, `SA_ONESHOT=SA_RESETHAND`.

REQ-8: `SA_UNSUPPORTED` is a feature-detection sentinel: it MUST be reflected back via `oldact` from `sigaction(2)`, allowing userspace to determine whether the kernel preserves all `sa_flags` bits.

REQ-9: `SA_RESTORER == 0x04000000` MUST NOT be reused for new flags on any new arch (compatibility hard-block).

REQ-10: `SIG_DFL=0`, `SIG_IGN=1`, `SIG_ERR=-1` (as `__sighandler_t` pointers).

REQ-11: `SIG_BLOCK=0`, `SIG_UNBLOCK=1`, `SIG_SETMASK=2`.

REQ-12: `struct sigaltstack` field order MUST be `ss_sp, ss_flags, ss_size`. `ss_flags` valid bits: `SS_ONSTACK=1`, `SS_DISABLE=2`, `SS_AUTODISARM=0x80000000`. Other bits MUST return `-EINVAL` from `sigaltstack(2)`.

REQ-13: `MINSIGSTKSZ` / `SIGSTKSZ`: generic minimum 2048 / recommended 8192. Per-arch override permitted (arm64 with SVE: `MINSIGSTKSZ` grows with vector length).

REQ-14: `siginfo_t` total size MUST be 128 bytes on LP64 (`__SI_MAX_SIZE`). Per arch, `__ARCH_SI_PREAMBLE_SIZE` reorders the leading 3 ints (`si_signo`, `si_errno`, `si_code`) — mips swaps `si_code` and `si_errno`.

REQ-15: Source codes MUST match: `SI_USER=0`, `SI_KERNEL=0x80`, `SI_QUEUE=-1`, `SI_TIMER=-2`, `SI_MESGQ=-3`, `SI_ASYNCIO=-4`, `SI_SIGIO=-5`, `SI_TKILL=-6`. `SI_FROMUSER` ⟺ `si_code <= 0`.

REQ-16: Per-signal `si_code` arms MUST match the documented numbering above. Synchronous fault signals (`SIGSEGV`, `SIGBUS`, `SIGILL`, `SIGFPE`, `SIGTRAP`) MUST carry a positive (kernel-origin) si_code and populate `si_addr` with the faulting user-space VA.

REQ-17: `SIGCHLD` `si_code` arms (`CLD_EXITED=1 .. CLD_CONTINUED=6`) MUST populate `si_pid`, `si_uid`, `si_status`, `si_utime`, `si_stime`.

REQ-18: `SIGSYS` from seccomp MUST carry `si_code = SYS_SECCOMP=1`, plus `si_call_addr`, `si_syscall`, `si_arch`, `si_errno` (return value).

REQ-19: `rt_sigqueueinfo(2)` MUST reject `siginfo->si_code >= 0` from a non-CAP_KILL process targeting another process — userspace cannot spoof a kernel-origin si_code.

REQ-20: `pidfd_send_signal(2)` MUST accept `siginfo == NULL` (synthesize SI_USER); a non-NULL siginfo with `si_code > 0` MUST require CAP_SYS_ADMIN or sender == receiver.

REQ-21: `sigevent` `sigev_notify` MUST be one of `{SIGEV_SIGNAL, SIGEV_NONE, SIGEV_THREAD, SIGEV_THREAD_ID}`. `SIGEV_MAX_SIZE == 64`.

REQ-22: `kill(pid, signo)` with `signo == 0` MUST perform permission-check only (no delivery). Negative `pid` targets process group `-pid`.

REQ-23: `tgkill(tgid, tid, signo)` MUST require `tid` belongs to `tgid`; otherwise `-ESRCH`.

REQ-24: `sigprocmask` "how" must be `SIG_BLOCK | SIG_UNBLOCK | SIG_SETMASK`; other values MUST return `-EINVAL`.

REQ-25: `rt_sigaction(2)` `sigsetsize` parameter MUST equal `sizeof(sigset_t)` (`_NSIG_WORDS * sizeof(long)`); otherwise `-EINVAL`.

## Acceptance Criteria

- [ ] AC-1: `_NSIG == 64`; `sizeof(sigset_t) == _NSIG/8` (8 bytes LP64).
- [ ] AC-2: `SIGKILL == 9`; `sigaction(SIGKILL, &new, NULL)` returns `-EINVAL`.
- [ ] AC-3: `SIGSTOP == 19`; `sigprocmask(SIG_BLOCK, &mask, NULL)` with SIGSTOP in mask: SIGSTOP remains deliverable.
- [ ] AC-4: `SIGRTMIN == 32`, `SIGRTMAX == 64`; 32 RT signals queue in order.
- [ ] AC-5: `sa_handler == SIG_IGN` for SIGCHLD: children don't become zombies.
- [ ] AC-6: `SA_RESETHAND`: handler reset to SIG_DFL after first delivery.
- [ ] AC-7: `SA_NODEFER`: signal not blocked during its own handler.
- [ ] AC-8: `SA_SIGINFO`: handler receives `(int, siginfo_t *, void *)`.
- [ ] AC-9: `SA_RESTART`: `read(2)` resumed across delivery.
- [ ] AC-10: `SA_UNSUPPORTED`: bit reflected back via `oldact`.
- [ ] AC-11: `SIGSEGV` from faulting load: `si_code == SEGV_MAPERR` (unmapped) or `SEGV_ACCERR` (denied).
- [ ] AC-12: `SIGCHLD` from exit: `si_code == CLD_EXITED`, `si_status == exit_code`.
- [ ] AC-13: `SIGSYS` from seccomp: `si_code == SYS_SECCOMP`, `si_syscall == nr`.
- [ ] AC-14: `tgkill(tgid_a, tid_in_tgid_b, SIGUSR1)` returns `-ESRCH`.
- [ ] AC-15: `kill(pid, 0)` performs permission check only; returns 0 if alive + permitted.
- [ ] AC-16: `sigaltstack({ss_sp, SS_ONSTACK, SIGSTKSZ}, NULL)`: handler with `SA_ONSTACK` runs on alt stack.
- [ ] AC-17: `sizeof(siginfo_t) == 128` on LP64.
- [ ] AC-18: `rt_sigqueueinfo` with spoofed `si_code = SI_KERNEL` from unprivileged returns `-EPERM`.
- [ ] AC-19: `MINSIGSTKSZ`, `SIGSTKSZ` are 2048 and 8192 on generic; per-arch overrides honored.
- [ ] AC-20: `pidfd_send_signal(pidfd, SIGTERM, NULL, 0)` delivers SIGTERM with `si_code == SI_USER`.

## Architecture

```
#[repr(C)]
#[derive(Copy, Clone, Default)]
pub struct Sigset {
    pub sig: [u64; _NSIG_WORDS],   // _NSIG_WORDS = 1 on LP64, 2 on ILP32 with [u32; 2]
}

pub type SignalHandler = Option<extern "C" fn(c_int)>;
pub type SigRestorer   = Option<extern "C" fn()>;

#[repr(C)]
pub struct Sigaction {
    pub sa_handler: SignalHandler,   // also reinterpreted as sa_sigaction when SA_SIGINFO
    pub sa_flags:   c_ulong,
    #[cfg(arch_has_sa_restorer)]
    pub sa_restorer: SigRestorer,
    pub sa_mask:    Sigset,          // LAST
}

#[repr(C)]
pub struct Sigaltstack {
    pub ss_sp:    UserPtr<c_void>,
    pub ss_flags: c_int,
    pub ss_size:  KernelSizeT,
}
pub type StackT = Sigaltstack;

#[repr(C)]
pub union SigvalT {
    pub sival_int: c_int,
    pub sival_ptr: UserPtr<c_void>,
}

#[repr(C)]
pub struct Siginfo {
    pub si_signo: c_int,
    pub si_errno: c_int,
    pub si_code:  c_int,
    /* + per-signal union, padded to __SI_MAX_SIZE (128 bytes on LP64) */
    pub _payload: [u8; 128 - 3 * core::mem::size_of::<c_int>()],
}

#[repr(C)]
pub struct Sigevent {
    pub sigev_value:  SigvalT,
    pub sigev_signo:  c_int,
    pub sigev_notify: c_int,            // SIGEV_*
    pub _sigev_un:    [u8; 64 - __ARCH_SIGEV_PREAMBLE_SIZE],
}

bitflags! {
    pub struct SaFlags : u32 {
        const NOCLDSTOP     = 0x0000_0001;
        const NOCLDWAIT     = 0x0000_0002;
        const SIGINFO       = 0x0000_0004;
        const UNSUPPORTED   = 0x0000_0400;
        const EXPOSE_TAGBITS= 0x0000_0800;
        const ONSTACK       = 0x0800_0000;
        const RESTART       = 0x1000_0000;
        const NODEFER       = 0x4000_0000;
        const RESETHAND     = 0x8000_0000;
    }
}
```

`SysSignal::sys_rt_sigaction(signo, act_user, oldact_user, sigsetsize) -> Result<()>`:
1. /* Strict size check */
2. if sigsetsize != sizeof::<Sigset>(): return Err(EINVAL).
3. /* Block uncatchable */
4. if signo == SIGKILL || signo == SIGSTOP: return Err(EINVAL).
5. if signo < 1 || signo > _NSIG: return Err(EINVAL).
6. /* Copy in */
7. let act = if !act_user.is_null() { Some(copy_from_user(act_user)?) } else { None };
8. /* Validate sa_flags */
9. if let Some(a) = &act {
       let known = SaFlags::all();
       /* SA_UNSUPPORTED: reflect, but don't reject */
       if a.sa_flags & !known.bits() & !arch_reserved_mask() != 0 { /* drop unknowns silently */ }
   }
10. /* Lock and swap */
11. spin_lock(&current.sighand.action_lock).
12. let old = current.sighand.action[signo - 1].clone();
13. if let Some(a) = act { current.sighand.action[signo - 1] = a; }
14. spin_unlock.
15. /* Copy out old */
16. if !oldact_user.is_null() { copy_to_user(oldact_user, &old)? };
17. return Ok(()).

`SysSignal::sys_rt_sigprocmask(how, set_user, oldset_user, sigsetsize) -> Result<()>`:
1. if sigsetsize != sizeof::<Sigset>(): return Err(EINVAL).
2. let old = current.blocked;
3. if !set_user.is_null() {
       let set = copy_from_user(set_user)?;
       /* SIGKILL, SIGSTOP cannot be blocked */
       let safe = set.without(SIGKILL).without(SIGSTOP);
       match how {
           SIG_BLOCK   => current.blocked = old.union(safe),
           SIG_UNBLOCK => current.blocked = old.difference(safe),
           SIG_SETMASK => current.blocked = safe,
           _           => return Err(EINVAL),
       }
   }
4. if !oldset_user.is_null() { copy_to_user(oldset_user, &old)?; }
5. /* Re-evaluate pending */
6. recalc_sigpending().
7. return Ok(()).

`SysSignal::sys_sigaltstack(uss, uoss) -> Result<()>`:
1. if !uoss.is_null() { copy_to_user(uoss, &current.sas_ss())?; }
2. if !uss.is_null() {
       let ss = copy_from_user(uss)?;
       /* Reject unknown flags */
       if ss.ss_flags & !(SS_DISABLE | SS_AUTODISARM) != 0 { return Err(EINVAL); }
       if ss.ss_flags & SS_DISABLE != 0 {
           current.set_sas_ss(0, 0, SS_DISABLE);
       } else {
           if ss.ss_size < MINSIGSTKSZ { return Err(ENOMEM); }
           current.set_sas_ss(ss.ss_sp, ss.ss_size, ss.ss_flags & SS_AUTODISARM);
       }
   }
3. return Ok(()).

`SysSignal::sys_pidfd_send_signal(pidfd, signo, info_user, flags) -> Result<()>`:
1. if flags != 0: return Err(EINVAL).
2. if signo < 1 || signo > _NSIG: return Err(EINVAL).
3. let target = pidfd_lookup(pidfd)?;
4. let info = if !info_user.is_null() {
       let i = copy_from_user(info_user)?;
       /* Forbid spoofing kernel-origin si_code */
       if i.si_code >= 0 && !cap_kill_allowed(current, target) {
           return Err(EPERM);
       }
       i
   } else {
       Siginfo::synthesize_user(signo, current)
   };
5. send_signal(target, signo, &info)?.
6. return Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sigset_size_matches_arch` | INVARIANT | `sizeof::<Sigset>() == _NSIG / 8`. |
| `sigaction_no_kill_no_stop` | INVARIANT | per-sigaction: signo ∈ {SIGKILL, SIGSTOP} ⟹ EINVAL. |
| `sigaction_sigsetsize_exact` | INVARIANT | per-rt_sigaction: sigsetsize != sizeof(Sigset) ⟹ EINVAL. |
| `sigprocmask_how_valid` | INVARIANT | per-rt_sigprocmask: how ∈ {0,1,2} ⟹ proceed, else EINVAL. |
| `sigprocmask_kill_stop_unblockable` | INVARIANT | per-rt_sigprocmask: SIGKILL/SIGSTOP cleared from blocked. |
| `sigaltstack_flags_valid` | INVARIANT | per-sigaltstack: ss_flags & ~(SS_DISABLE|SS_AUTODISARM) ⟹ EINVAL. |
| `sigaltstack_size_minimum` | INVARIANT | per-sigaltstack: ss_size < MINSIGSTKSZ ∧ !DISABLE ⟹ ENOMEM. |
| `pidfd_send_signal_no_spoof` | INVARIANT | per-pidfd_send_signal: si_code >= 0 unprivileged ⟹ EPERM. |
| `rt_sigqueueinfo_no_spoof` | INVARIANT | per-rt_sigqueueinfo: cross-process si_code >= 0 unprivileged ⟹ EPERM. |
| `signo_in_range` | INVARIANT | per-syscall: signo ∈ [1, _NSIG] else EINVAL. |
| `sigaction_layout_arch` | INVARIANT | per-arch: sa_mask is LAST field; sa_restorer presence matches arch. |
| `siginfo_size_128` | INVARIANT | `sizeof::<Siginfo>() == 128` on LP64. |

### Layer 2: TLA+

`uapi/headers/signal.tla`:
- Per-`sigaction(2)` install → signal-raised → handler-invoked → mask-restored cycle.
- Per-`sigaltstack(2)` switch on `SA_ONSTACK` delivery.
- Per-`SIGRTMIN..SIGRTMAX` queueing-order property.
- Properties:
  - `safety_SIGKILL_uncatchable` — per-thread: SIGKILL ⟹ termination (no handler invocation).
  - `safety_SIGSTOP_unblockable` — per-thread: SIGSTOP not in blocked set on delivery.
  - `safety_RT_signals_queue_in_order` — per-RT-pair: lower-numbered delivered first.
  - `safety_sa_resethand_one_shot` — per-handler: SA_RESETHAND ⟹ next delivery uses SIG_DFL.
  - `safety_sigaltstack_one_at_a_time` — per-thread: SS_ONSTACK during handler ⟹ recursive entry uses extended frame.
  - `safety_pidfd_signal_no_pid_reuse` — per-pidfd: kill targets exact original task, never PID-reused.
  - `liveness_pending_signal_eventually_delivered` — per-unblocked-signal: arrived ⟹ eventually handled.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Sigset` layout per arch matches `_NSIG_WORDS * sizeof::<c_ulong>()` | `Sigset` |
| `Sigaction` layout: handler/flags/[restorer]/mask LAST | `Sigaction` |
| `Sigaltstack` layout: ss_sp / ss_flags / ss_size | `Sigaltstack` |
| `Siginfo` layout: signo/errno/code preamble + 128B total | `Siginfo` |
| `Sigevent` layout: 64B total, preamble matches `__ARCH_SIGEV_PREAMBLE_SIZE` | `Sigevent` |
| `sys_rt_sigaction` post: SIGKILL/SIGSTOP unaffected | `sys_rt_sigaction` |
| `sys_rt_sigprocmask` post: blocked-mask sanitized | `sys_rt_sigprocmask` |
| `sys_pidfd_send_signal` post: si_code spoof rejected | `sys_pidfd_send_signal` |

### Layer 4: Verus/Creusot functional

`Per rt_sigaction(2) → validate signo/sigsetsize → swap sighand action → reflect oldact` semantic equivalence: per-`rt_sigaction(2)` man page and per-`kernel/signal.c` upstream. `Per kill(2)/tgkill(2)/pidfd_send_signal(2) → permission check → enqueue or coalesce → wake target` semantic equivalence: per-POSIX 1003.1 §2.4 + Linux real-time extensions.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

Signal UAPI reinforcement:

- **`SIGKILL` / `SIGSTOP` uncatchable enforced at sigaction entry** — defense against per-handler-install bypass.
- **`SIGKILL` / `SIGSTOP` unblockable enforced at sigprocmask entry** — defense against per-mask-block bypass.
- **Strict `sigsetsize` check** — defense against per-undersized-copy disclosure / overflow.
- **`SA_UNSUPPORTED` reflection** — userspace can detect kernel feature support without trial-and-error.
- **`SS_FLAG_BITS` strict mask** — defense against per-future-flag silent activation in `sigaltstack`.
- **`MINSIGSTKSZ` enforced** — defense against per-tiny-alt-stack overflow during signal delivery.
- **`pidfd_send_signal` permission check on pidfd not pid** — defense against per-PID-reuse signal-to-wrong-process.
- **`rt_sigqueueinfo` cross-process `si_code` spoof check** — defense against per-fake-kernel-origin sigqueue.
- **Per-thread vs. per-process target distinction strict** — `tgkill` requires `tid ∈ tgid` or `-ESRCH`.
- **`SA_NODEFER + SA_RESETHAND` combined: kernel restores SIG_DFL atomically with handler entry** — defense against per-reentry-race.
- **`SA_SIGINFO` siginfo bounds-validated** — defense against per-siginfo-payload OOB read in handler.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF/USERCOPY** on every `copy_from_user(act, mask, info)` and `copy_to_user(oldact, oldmask)` — defense against per-syscall user-pointer abuse and per-kernel-data-leakage in `oldact` / `oldmask` copy paths.
- **PAX_RANDKSTACK on syscall entry** — randomize kernel-stack offset per `rt_sigaction`, `rt_sigprocmask`, `pidfd_send_signal` entry so any disclosure of stack-resident sighand or task pointers (e.g., a siginfo leak) is statistically useless.
- **GRKERNSEC_BRUTE** — repeated SIGSEGV/SIGBUS/SIGFPE crashes from the same uid within a short window trigger `SIGKILL` and a forkbomb-resistant cooldown. Defeats brute-force ASLR / stack-canary scans by making each crash count toward the per-uid budget.
- **GRKERNSEC_SIGNALS** — non-root uid B may not send any non-`SIGTERM`-class signal to a setuid binary owned by uid A; spoofing `SI_KERNEL` via `rt_sigqueueinfo` is blocked even for CAP_KILL holders if the target is gated by `proc_id_perm`.
- **PaX `kill`-via-`ptrace` gate** — a debugger attached to a process cannot install a handler that survives detach if PAX MPROTECT is active for that binary (prevents post-detach JIT-trampoline injection via `sa_restorer`).
- **`sa_restorer` validated against `executable VMA`** — kernel refuses to install a `sa_restorer` pointing outside an executable VMA owned by the same mm. Stops the classic "point sa_restorer at gadget" ROP-bootstrap technique.
- **`SI_KERNEL` (`si_code = 0x80`) forging blocked for ANY userspace sender** — even with `CAP_KILL` / `CAP_SYS_ADMIN`, only the kernel itself synthesizes `SI_KERNEL`. `rt_sigqueueinfo` with `si_code = SI_KERNEL` always returns `-EPERM`.
- **`SIGEV_THREAD_ID` validated cgroup-scope** — `_tid` must be in the sender's PID namespace / cgroup; cross-namespace tid-targeted timers return `-ESRCH` (avoids per-namespace-escape timer-side-channel).
- **GRKERNSEC_PROC_USER** — `/proc/<pid>/status` `SigBlk`, `SigIgn`, `SigCgt` masks are hidden from non-same-uid readers, eliminating the "scan target's signal disposition" reconnaissance step.
- **`SIGFPE`/`SIGSEGV` `si_addr` tag-bits stripped** unless `SA_EXPOSE_TAGBITS` is set — defense against ARM MTE / sparc ADI tag-bit leakage through fault delivery to unaware handlers.
- **`signalfd4(2)` cgroup-scoped read** — a signalfd inherited across container boundaries cannot read signals destined for the host namespace; the kernel filters during read.
- **`MINSIGSTKSZ` enforced AT install AND at delivery** — if an SVE-enabled process changes vector length after `sigaltstack`, kernel re-checks the alt-stack size at signal delivery and aborts to `SIG_DFL` (kill) rather than overflow.

## Open Questions

(none at this Tier-5 level — `kernel/signal.c`, `kernel/signalfd.c` Tier-3 docs cover per-syscall internals)

## Out of Scope

- `include/uapi/asm/sigcontext.h` per-arch register saves on signal frame (covered per-arch in `arch/<arch>/uapi`)
- `include/uapi/linux/signalfd.h` `signalfd_siginfo` userspace struct (covered in `uapi/headers/signalfd.md` Tier-5)
- `include/uapi/asm-generic/poll.h` POLL_* values overlap (covered in `uapi/headers/poll.md` Tier-5)
- `kernel/signal.c` / `kernel/signalfd.c` / `kernel/ptrace.c` implementations — covered in `kernel/00-overview.md` Tier-3
- `arch/*/kernel/signal.c` per-arch sigframe setup — covered per-arch
- POSIX timers `timer_create(2)` (covered in `uapi/syscalls/posix-timer.md` Tier-5)
- Implementation code
