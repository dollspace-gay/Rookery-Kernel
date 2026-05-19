---
title: "Tier-5 syscall: fork(2) — syscall 57"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`fork(2)` is the historical POSIX task-creation primitive: it creates a new
process that is an almost-exact copy of the caller (same address space
contents via copy-on-write, same open files, same fs context, same signal
dispositions, same cwd, same umask, same controlling tty). On Linux it is
strictly a thin shim over `kernel_clone` with the flag set
`flags = SIGCHLD` — exactly one flag bit (`CSIGNAL` = `SIGCHLD = 17`) is
asserted, no `CLONE_*` resource-share bits are set, and no namespace bits
are set.

`fork(2)` is the bedrock of the Unix process model: every shell pipeline
(`bash`, `zsh`), every traditional daemon (`postfix`, `apache2-prefork`),
every `system(3)`, every CGI / FastCGI worker, every `Perl`/`Ruby`/`Python`
multi-process server reaches `fork`. New code generally prefers
`posix_spawn(3)` (which uses `vfork`/`clone3` under the hood) or
`clone3(2)` directly, but `fork` MUST remain ABI-stable forever.

This syscall is **not present on every architecture**. The "generic system
calls" archs (`arm64`, `riscv`, `loongarch`, modern `s390x`) wire libc's
`fork()` to `clone(SIGCHLD, ...)` in user space and omit a kernel-side
`__NR_fork`. The x86 family, ppc, sparc, mips, alpha, parisc, m68k, sh,
xtensa, microblaze KEEP a dedicated `__NR_fork` for legacy programs. Both
forms MUST behave identically.

This Tier-5 covers `kernel/fork.c::SYSCALL_DEFINE0(fork, void)` (~10 lines).

### Acceptance Criteria

- [ ] AC-1: `fork()` returns child pid in parent and 0 in child.
- [ ] AC-2: On x86_64, syscall number is 57.
- [ ] AC-3: On arm64, direct `syscall(__NR_fork)` (no such number) returns `-ENOSYS`; libc `fork()` succeeds via `clone(SIGCHLD)`.
- [ ] AC-4: Child has a new mm — write in child does not affect parent (COW).
- [ ] AC-5: Child has independent fd offsets after lseek in child.
- [ ] AC-6: Child has independent cwd/umask after chdir/umask in child.
- [ ] AC-7: Child sees same sig dispositions; SIG_IGN inherited.
- [ ] AC-8: Child's pending signals are empty.
- [ ] AC-9: Child exits → parent receives `SIGCHLD` with `si_code=CLD_EXITED`.
- [ ] AC-10: `RLIMIT_NPROC` exceeded → `-EAGAIN` for non-CAP_SYS_RESOURCE caller.
- [ ] AC-11: Child's `getppid()` returns the caller's pid.
- [ ] AC-12: Child's CPU times start at 0.
- [ ] AC-13: Child does NOT inherit `PR_SET_PDEATHSIG`.
- [ ] AC-14: Child shares same uid / gid / capability bounding set / `no_new_privs` / seccomp filters as parent.
- [ ] AC-15: Auditd records the new pid.
- [ ] AC-16: `vfork(2)` and `fork(2)` differ ONLY in `CLONE_VM | CLONE_VFORK` — verify by tracing.
- [ ] AC-17: Forked child sharing controlling tty: parent and child both receive `SIGINT` on `^C`.

### Architecture

```rust
#[syscall(nr = 57, abi = "sysv", arches = "x86_64,i386,powerpc,sparc,mips,s390x,alpha,parisc,m68k,sh,xtensa,microblaze")]
pub fn sys_fork() -> isize {
    let args = KernelCloneArgs {
        flags:        0,
        exit_signal:  SIGCHLD as u32,
        pidfd:        0,
        child_tid:    UserPtr::null(),
        parent_tid:   UserPtr::null(),
        stack:        UserPtr::null(),
        stack_size:   0,
        tls:          0,
        set_tid:      UserPtr::null(),
        set_tid_size: 0,
        cgroup:       0,
    };
    Fork::kernel_clone(args)
}
```

`Fork::kernel_clone(args)` is the shared workhorse — see clone.md
architecture section.

Per-arch wiring:
- x86_64 syscall_64.tbl row `57  common  fork  sys_fork`.
- i386 syscall_32.tbl row `2  i386  fork  sys_fork`.
- arm64 / riscv / loongarch: NO row; deliberate.

### Out of Scope

- `kernel_clone` / `copy_process` task duplication (Tier-3 `kernel/fork.c`).
- `vfork(2)` syscall (separate Tier-5 doc).
- `clone(2)` / `clone3(2)` syscalls (separate Tier-5 docs).
- COW page-fault handling (Tier-3 `mm/memory.c` `do_wp_page`).
- Implementation code.

### signature

```c
long fork(void);
```

No arguments. Returns twice — once in the parent (with child pid), once in
the child (with 0).

### parameters

(none)

### return value

| Value | Meaning |
|---|---|
| `> 0` (in parent) | Child PID in caller's pid_ns. |
| `0`   (in child)  | Successful fork — child execution begins here. |
| `-1` + `errno`    | Failure (no child created); see Errors. |

### errors

| errno | Trigger |
|---|---|
| `EAGAIN` | `RLIMIT_NPROC` exceeded for caller's real uid. |
| `EAGAIN` | System-wide `task_struct` slab cap reached. |
| `ENOMEM` | Page-tables / kernel stack allocation failed. |
| `ENOSYS` | Architecture does not implement `__NR_fork` (e.g. arm64 generic-sys; libc emulates via `clone`). |
| `ERESTARTNOINTR` | Signal received during pre-copy phase. |

`fork(2)` does NOT return `-EINVAL`: there are no caller-supplied arguments
that could be invalid.

### abi surface

```text
__NR_fork (x86_64)  = 57
__NR_fork (i386)    = 2
__NR_fork (powerpc) = 2
__NR_fork (s390x)   = 2     (compat-only on modern s390x)
__NR_fork (sparc)   = 2
__NR_fork (mips)    = 4002 / 5057 (O32 / N64)
__NR_fork (alpha)   = 2
__NR_fork (parisc)  = 2
__NR_fork (arm64)   = NOT WIRED — libc must use clone(SIGCHLD, ...)
__NR_fork (riscv)   = NOT WIRED — libc must use clone(SIGCHLD, ...)
__NR_fork (loongarch) = NOT WIRED — libc must use clone(SIGCHLD, ...)
```

Effective flag set:

```text
fork() ≡ clone(SIGCHLD, NULL, NULL, NULL, 0)
       ≡ kernel_clone(KernelCloneArgs {
             flags: 0,
             exit_signal: SIGCHLD,
             stack: NULL,
             ...
         })
```

### compatibility contract

REQ-1: The syscall number is **57** on x86_64. Per-arch numbers are
architecture-stable forever. Where a number is published, removing it is an
ABI break.

REQ-2: On archs where `__NR_fork` is wired, the syscall MUST behave exactly
as `clone(SIGCHLD, NULL, NULL, NULL, 0)`.

REQ-3: On archs without `__NR_fork` (arm64, riscv, loongarch), `fork(2)` is a
libc-only construct: libc emits `clone(SIGCHLD, ...)`. The kernel has no
`__NR_fork` entry; an attempted direct invocation (e.g. via `syscall(2)` with
a hard-coded number) MUST return `-ENOSYS`.

REQ-4: Child task properties:
- Copy of caller's address space (COW; pages mark `_PAGE_RW` cleared on both
  sides, restored on first write fault).
- New `mm_struct` (NOT shared — `CLONE_VM` is 0).
- New `files_struct` cloned from caller (NOT shared — `CLONE_FILES` is 0).
  All fds are duplicated; offsets are independent thereafter.
- New `fs_struct` cloned (NOT shared — `CLONE_FS` is 0). cwd, root, umask
  copied; subsequent `chdir`/`chroot`/`umask` in child does not affect parent.
- New `sighand_struct` cloned (NOT shared — `CLONE_SIGHAND` is 0). All
  dispositions copied as-is.
- New `signal_struct` (NOT shared — `CLONE_THREAD` is 0). Pending signals
  cleared in the child (per POSIX `fork(2)`).
- Same uid / gid / capabilities / no_new_privs / seccomp filters as caller.
- Same cgroup, cpuset, IO context as caller (NOT new namespaces).
- Same controlling tty, session, process group.
- Parent pid = caller's pid (NOT `CLONE_PARENT`).

REQ-5: Exit notification: when the child exits, `SIGCHLD` is queued to the
parent (unless parent has `SA_NOCLDWAIT` or `SIG_IGN` for `SIGCHLD`). The
signal carries `siginfo` with `si_code = CLD_EXITED` / `CLD_KILLED` /
`CLD_DUMPED`.

REQ-6: `RLIMIT_NPROC` is enforced against the caller's real-uid process
count; `CAP_SYS_ADMIN` or `CAP_SYS_RESOURCE` bypasses.

REQ-7: Child's CPU times reset to 0; `rusage` zeroed.

REQ-8: Child's pending timers (POSIX timers, alarm) NOT inherited; itimers
in `ITIMER_REAL/_VIRTUAL/_PROF` ARE inherited.

REQ-9: Child's mlock/mlockall state is NOT inherited (per POSIX).

REQ-10: Child's `prctl(PR_SET_NAME)` value IS inherited.

REQ-11: Child's `PR_SET_PDEATHSIG` is CLEARED in the child (parent-death
signal is per-process and does not transit fork).

REQ-12: Child inherits the parent's robust-futex list pointer (set to NULL
in the child until userspace re-registers).

REQ-13: Child inherits SysV semaphore-undo lists CLONED (not shared — that
would require `CLONE_SYSVSEM`).

REQ-14: Child inherits the parent's audit context, but a fresh
`audit_context.serial` is allocated.

REQ-15: Child inherits parent's `oom_score_adj`.

REQ-16: Child inherits parent's `prctl(PR_GET_CHILD_SUBREAPER)` flag and
the subreaper hierarchy; if a parent is a subreaper, the child's
adoption-on-orphan target is the nearest subreaper ancestor.

REQ-17: Audit: a successful `fork` MUST emit `AUDIT_SYSCALL` with the
new pid; namespace IDs are unchanged (none created).

REQ-18: `fork(2)` is **not** allowed to be retried by libc on `EINTR`; the
syscall is uninterruptible past the copy_process commit point. Pre-commit
interruption returns `-ERESTARTNOINTR` (kernel-internal; userspace sees
`-EINTR` only if the signal handler is `SA_RESTART=0`).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fork_flag_set_is_only_csignal_sigchld` | INVARIANT | `sys_fork`: KernelCloneArgs.flags == 0; exit_signal == SIGCHLD. |
| `fork_no_user_pointers` | INVARIANT | All UserPtr fields are null. |
| `fork_returns_pid_or_zero_or_negerrno` | INVARIANT | Return ∈ {pid>0, 0, -EAGAIN, -ENOMEM, -ENOSYS, -ERESTARTNOINTR}. |

### Layer 2: TLA+

`kernel/fork-syscall.tla`:
- States: PARENT_READY → COPY_PROCESS → PID_ALLOC → WAKE_CHILD → RETURN(pid)/RETURN(0).
- Properties:
  - `safety_two_returns` — exactly one parent return and one child return.
  - `safety_pid_unique` — child pid not currently in use in caller's pid_ns.
  - `liveness_child_runs` — child reaches userspace eventually.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_fork` post: KernelCloneArgs literal-equivalent to `clone(SIGCHLD, ...)` | `sys_fork` |
| `copy_process(args)` semantics-equivalent to fork-via-clone | `copy_process` |
| Pending signals empty in child | `Task::copy_signal` |

### Layer 4: Verus / Creusot functional

POSIX `fork(2)` semantic equivalence: every clause of POSIX 1003.1 fork()
realized. LTP `fork01..fork14` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fork(2)` reinforcement:

- **Per-`RLIMIT_NPROC` strict** — defense against per-fork-bomb.
- **Per-`SIGCHLD` queued from kernel-source** — defense against per-spoofed-exit-notification (`si_code` cannot be forged).
- **Per-`PR_SET_PDEATHSIG` cleared in child** — defense against per-pdeathsig leak across new process.
- **Per-pending-signal clear in child** — defense against per-pre-fork-signal carry.
- **Per-mm COW page-table strict** — defense against per-COW-bypass (e.g. CVE-2022-0847 Dirty Pipe class) via _PAGE_RW invariants.
- **Per-mlock not inherited** — defense against per-large-mlock fork amplification.
- **Per-audit record** — defense against per-process-creation elision.
- **Per-arch absence (`arm64`/`riscv`/`loongarch`)** — defense against per-legacy-ABI surface bloat (libc handles it).

### grsecurity / pax surface

- **PAX_RANDKSTACK at syscall entry** — `sys_fork` randomizes kernel stack
  offset; combined with COW page-table setup, defeats per-fork stack-layout
  prediction for ROP.
- **PaX UDEREF** — `sys_fork` has no user pointers (no buffers), but the
  follow-on `wake_up_new_task` path that copies regs to child still observes
  UDEREF semantics for any subsequent user write (e.g. trampoline restore).
- **GRKERNSEC_HARDEN_PTRACE** — ptrace state is inherited per the
  yama-tightened policy: a child of a ptraced parent does NOT itself become
  ptraced unless `PTRACE_TRACEME` is re-invoked.
- **GRKERNSEC_PROC restrictions** — newly forked pid is published into
  `/proc/<pid>` only after per-uid/per-group hide-filter is applied.
- **GRKERNSEC_CHROOT** — `fork(2)` is permitted in chroot (it does not create
  namespaces) but the resulting child inherits the chroot lockdown.
- **GRKERNSEC_BRUTE** — `fork()` returning `-EAGAIN` due to `RLIMIT_NPROC` is
  counted into the per-uid forkbomb detector; sustained over threshold →
  per-uid lockout (deny new clone/fork/execve for N seconds).
- **GRKERNSEC_SIGNALS spoof prevention** — `SIGCHLD` queued on child exit is
  kernel-sourced; `si_code = CLD_EXITED`; cannot be spoofed via `SI_USER`.
- **Per-grsec `chroot_findtask`** — fork inside chroot does NOT enable
  child to see parent's siblings outside the chroot via `/proc`.
- **Per-grsec `dmesg` restriction** — fork-rate spikes logged to dmesg are
  rate-limited; only `CAP_SYSLOG` may read them.

