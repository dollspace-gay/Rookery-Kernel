# Tier-5 syscall: clone(2) — syscall 56

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/fork.c (sys_clone, kernel_clone, copy_process)
  - include/linux/syscalls.h (SYSCALL_DEFINE5(clone, ...))
  - include/uapi/linux/sched.h (CLONE_* flags, CSIGNAL)
  - arch/x86/entry/syscalls/syscall_64.tbl (56  common  clone)
  - arch/arm64/include/uapi/asm/unistd.h (NR 220 — generic-sys map of __NR_clone)
-->

## Summary

`clone(2)` is the primary task-creation primitive on Linux. Every thread library
(`pthread_create`, `std::thread`), every container runtime, every `posix_spawn`
fast-path, and every userspace process supervisor ultimately reaches this
syscall (or its successor `clone3(2)`). Unlike POSIX `fork(2)`, `clone(2)` lets
the caller select exactly which resources are shared with the child: virtual
memory (`CLONE_VM`), filesystem-context (`CLONE_FS`), open-file table
(`CLONE_FILES`), signal-disposition table (`CLONE_SIGHAND`), parent-process
identity (`CLONE_PARENT`), thread-group membership (`CLONE_THREAD`), and the
eight namespace families (`CLONE_NEW{NS,UTS,IPC,USER,PID,NET,CGROUP,TIME}`).
The low byte of the `flags` argument is the **`CSIGNAL`** field — the
signal-number the kernel will queue on the parent when the child exits
(`SIGCHLD` for `fork`-style semantics, `0` for thread-style).

The argument ordering of `clone(2)` is **architecture-dependent**: on x86_64 it
is `(flags, stack, parent_tid, child_tid, tls)`; on x86 (i386) it is
`(flags, stack, parent_tid, tls, child_tid)`; on s390 / cris it is
`(stack, flags, …)`. Rookery follows the upstream per-arch ordering exactly —
the syscall is a five-argument shim that re-orders into a canonical
`KernelCloneArgs` struct before calling the shared `kernel_clone` path. All new
code SHOULD prefer `clone3(2)`, which has a stable argument order.

This Tier-5 covers `kernel/fork.c::SYSCALL_DEFINE5(clone, ...)` (~60 lines of
direct entry-point code) and its interaction with `kernel_clone` /
`copy_process`. The full task-creation Tier-3 (`kernel/fork.c`) is a separate
doc.

## Signature

```c
/* x86_64 / aarch64 / riscv / loongarch: */
long clone(unsigned long flags,
           void *stack,
           int  *parent_tidptr,
           int  *child_tidptr,
           unsigned long tls);

/* x86 (i386), s390x, microblaze swap arg 4 and 5: */
long clone(unsigned long flags,
           void *stack,
           int  *parent_tidptr,
           unsigned long tls,
           int  *child_tidptr);
```

The libc-visible wrapper `__clone` is hand-written assembly per arch — it
sets up the child stack, invokes the syscall, and on the child branch jumps
to a caller-supplied function pointer with a single `void*` argument.

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `flags` | `unsigned long` | in | Bitwise OR of `CLONE_*` (high 24 bits) and `CSIGNAL` (low 8 bits). |
| `stack` | `void *` | in | Lowest address of child stack, OR `NULL` to share parent's stack (then the COW-on-write semantics of `CLONE_VM=0` apply, or the child uses the parent SP unchanged if `CLONE_VM=1` — i.e. `vfork` / `fork`). |
| `parent_tidptr` | `int *` | out | If `CLONE_PARENT_SETTID`: child's TID is written here in the parent's address space, BEFORE child runs. |
| `child_tidptr` | `int *` | in/out | If `CLONE_CHILD_SETTID`: TID written in child's address space at first run. If `CLONE_CHILD_CLEARTID`: zero'd at thread-exit and `FUTEX_WAKE` issued. |
| `tls` | `unsigned long` | in | If `CLONE_SETTLS`: new TLS register value (arch-specific: GS_BASE on x86_64, TPIDR_EL0 on aarch64, tp on riscv). |

## Return value

| Value | Meaning |
|---|---|
| `> 0` (in parent) | Child PID (or TID if `CLONE_THREAD`). |
| `0`   (in child)  | Successful clone — child execution begins here. |
| `-1` + `errno`    | Failure (no child created); see Errors. |

If `CLONE_PIDFD` is set, an additional out-parameter pidfd is written into
`*parent_tidptr` (yes, the same slot — this is the `clone(2)` quirk that
`clone3(2)` fixed by giving pidfd its own field). The pidfd is `O_CLOEXEC` and
suitable for `pidfd_send_signal(2)`, `waitid(P_PIDFD, …)`, and epoll.

## Errors

| errno | Trigger |
|---|---|
| `EAGAIN` | `RLIMIT_NPROC` or system `task_struct` slab cap reached. |
| `EINVAL` | Flag combination invalid (see REQ-2..REQ-7). |
| `EINVAL` | `CSIGNAL` byte names a signal not in `[0, _NSIG]`. |
| `EINVAL` | `CLONE_NEWTIME` set in low 32 bits via `clone(2)` (it overlaps `CSIGNAL`). |
| `ENOMEM` | Unable to allocate `task_struct`, page tables, or stack. |
| `ENOSPC` | Per-pid-namespace task limit reached. |
| `EPERM`  | `CLONE_NEW*` without `CAP_SYS_ADMIN` in owning user-ns (except `CLONE_NEWUSER` under `kernel.unprivileged_userns_clone=1`). |
| `EPERM`  | `CLONE_PIDFD` combined with `CLONE_DETACHED` (pre-5.2 historical; fully rejected). |
| `EUSERS` | Per-user-ns nesting limit (default 32) reached. |
| `EFAULT` | `parent_tidptr` or `child_tidptr` not writable / not aligned. |
| `ERESTARTNOINTR` | Signal received during pre-copy phase. |

## ABI surface (constants)

### `CSIGNAL` mask and exit-signal encoding

```text
CSIGNAL = 0x000000ff   /* low 8 bits of flags = signal to send to parent on exit */
```

The caller OR's the desired signal number directly into the low byte:
`clone(CLONE_VM | CLONE_FS | SIGCHLD, …)`. The signal MUST be `0`
(no exit notification, only legal in combination with `CLONE_THREAD`),
or `SIGCHLD` (POSIX default for fork-like semantics), or `1..63` for an
RT signal. Anything `>= _NSIG` is `-EINVAL`.

`CLONE_NEWTIME = 0x00000080` overlaps the `CSIGNAL` mask — it is therefore
**unreachable** via `clone(2)`'s `unsigned long flags` (because no caller can
both name a real exit signal and set bit 0x80 without ambiguity); the kernel
explicitly returns `-EINVAL` for any `clone(2)` call with the `0x80` bit set.
Use `clone3(2)` or `unshare(2)` to create a new time namespace.

### `CLONE_*` low-32 flags (complete table — see `uapi/sched.md` for hex values)

| Flag | Resource shared / new | Notes |
|---|---|---|
| `CLONE_VM` | VM (mm_struct, page tables) | Required by `CLONE_SIGHAND`. |
| `CLONE_FS` | filesystem-context (cwd, root, umask) | Mutually exclusive with `CLONE_NEWNS`/`CLONE_NEWUSER`. |
| `CLONE_FILES` | open-file table | Each fd's offset and flags shared. |
| `CLONE_SIGHAND` | signal-disposition table | Requires `CLONE_VM`. |
| `CLONE_PIDFD` | (out) pidfd to parent | Returns pidfd via `parent_tidptr`. |
| `CLONE_PTRACE` | tracer inherits | Subject to `CLONE_UNTRACED` from tracer. |
| `CLONE_VFORK` | parent suspends until child exec/exit | Wakeup via `vfork_done` completion. |
| `CLONE_PARENT` | child's parent = caller's parent | Forbidden when crossing pid-ns. |
| `CLONE_THREAD` | child joins caller's thread group | Requires `CLONE_SIGHAND` + `CLONE_VM`; `CSIGNAL` MUST be 0. |
| `CLONE_NEWNS` | new mount namespace | Needs `CAP_SYS_ADMIN`. |
| `CLONE_SYSVSEM` | SysV semaphore-undo list shared | Otherwise child gets fresh sem_undo. |
| `CLONE_SETTLS` | child's TLS register = `tls` | Arch-specific register. |
| `CLONE_PARENT_SETTID` | write TID to parent at `parent_tidptr` | Pre-run, in parent's mm. |
| `CLONE_CHILD_CLEARTID` | zero+wake `child_tidptr` at exit | Backbone of pthread_join. |
| `CLONE_DETACHED` | (ignored / no-op since 2.5.32) | Historical NPTL flag. |
| `CLONE_UNTRACED` | tracer canNOT force `CLONE_PTRACE` | Set by kernel-thread creators. |
| `CLONE_CHILD_SETTID` | write TID at `child_tidptr` in child mm | Post-run, in child's mm. |
| `CLONE_NEWCGROUP` | new cgroup namespace | Needs `CAP_SYS_ADMIN`. |
| `CLONE_NEWUTS` | new UTS namespace | Needs `CAP_SYS_ADMIN`. |
| `CLONE_NEWIPC` | new IPC namespace | Needs `CAP_SYS_ADMIN`; clears SysV. |
| `CLONE_NEWUSER` | new user namespace | Mutually exclusive with `CLONE_FS`; gates other `CLONE_NEW*`. |
| `CLONE_NEWPID` | new PID namespace | Forbidden with `CLONE_THREAD`. |
| `CLONE_NEWNET` | new network namespace | Needs `CAP_SYS_ADMIN`. |
| `CLONE_IO` | share IO context (CFQ/blk-cgroup) | Shared per-task ioc. |

## Compatibility contract

REQ-1: The syscall number is **56** on x86_64 (common) and **220** in
the generic syscall numbering (arm64, riscv, loongarch — where it is
named via `__NR_clone` resolving to the canonical generic number 220).
Architecture-specific tables MUST register at the correct slot.

REQ-2: `CLONE_THREAD` requires `CLONE_SIGHAND` AND `CLONE_VM`. The transitive
check is `(flags & CLONE_THREAD) ⟹ (flags & CLONE_SIGHAND) ⟹ (flags & CLONE_VM)`.
Violation MUST return `-EINVAL` before any task allocation.

REQ-3: `CLONE_THREAD` MUST have `CSIGNAL == 0` — thread exit cannot deliver
`SIGCHLD` to parent (the parent receives `SI_TKILL`-style notification via
`futex_wake` on `CLONE_CHILD_CLEARTID` instead). `CSIGNAL != 0` with
`CLONE_THREAD` MUST return `-EINVAL`.

REQ-4: `CLONE_NEWUSER | CLONE_FS` returns `-EINVAL`. Changing the user
namespace requires that the FS root and cwd not be shared with the parent
(otherwise root could leak across user-ns boundaries).

REQ-5: `CLONE_NEWPID | CLONE_THREAD` returns `-EINVAL`. A thread cannot enter
a new PID namespace — only a new process group leader can.

REQ-6: `CLONE_PIDFD | CLONE_THREAD` historically returned `-EINVAL` (a thread
pidfd was meaningless). Since the kernel-5.5 era this is **permitted in
`clone3(2)` only**; `clone(2)` with both flags MUST still return `-EINVAL`
for ABI compatibility.

REQ-7: `CLONE_PIDFD | CLONE_DETACHED` returns `-EINVAL` (was historically
`CLONE_PIDFD | CLONE_PARENT_SETTID` overload — pidfd write target collides).

REQ-8: `CLONE_NEW{NS,UTS,IPC,CGROUP,NET}` each require `CAP_SYS_ADMIN` in the
user namespace that owns the target namespace being unshared.
`CLONE_NEWUSER` is the **exception** — it may be created unprivileged iff
`sysctl kernel.unprivileged_userns_clone == 1` (Rookery default `0`; mainline
default `1`).

REQ-9: `flags & ~(CSIGNAL | all known CLONE_* low bits)` MUST return `-EINVAL`
(no silent ignoring of unknown bits, to keep the ABI sealed for `clone3`).

REQ-10: `CSIGNAL` byte: `0` (legal only with `CLONE_THREAD`), or
`1..(_NSIG-1)` legitimate signals. Out-of-range MUST return `-EINVAL`.

REQ-11: `parent_tidptr` writability: when `CLONE_PARENT_SETTID` OR `CLONE_PIDFD`
is set, the kernel MUST verify the pointer is writable (`access_ok` +
4-byte aligned + within mm bounds). Failure MUST return `-EFAULT` BEFORE
any task allocation.

REQ-12: `child_tidptr` writability and alignment are deferred to first child
run — but the pointer is captured in `task_struct.set_child_tid` /
`clear_child_tid` and validated lazily; a bad pointer at thread-exit
silently fails the futex wake (matches mainline behavior).

REQ-13: `stack == NULL` is permitted: child starts with caller's stack pointer
(legacy `fork`-style; only safe with `CLONE_VM=0` because COW protects writes).

REQ-14: Returning to userspace in child: child PC is the caller's PC; child SP
is `stack` (if non-NULL) or caller's SP. The return-value register holds 0
in the child, and the new pid/tid in the parent.

REQ-15: `kernel_clone()` is the shared workhorse — it accepts a
`struct kernel_clone_args` whose layout mirrors `clone3`'s `clone_args` plus
the legacy positional fields; `clone(2)` MUST translate its 5 args into this
struct identically to mainline.

REQ-16: `RLIMIT_NPROC` enforcement: pre-allocation check counts the user's
existing processes against the soft limit; `CAP_SYS_ADMIN` or `CAP_SYS_RESOURCE`
bypasses. Over-limit returns `-EAGAIN`.

REQ-17: Per-user-namespace nesting depth limit (`MAX_USER_NS_DEPTH = 32`).
Exceeded returns `-EUSERS`.

REQ-18: `CLONE_PIDFD`: the returned fd MUST be `O_CLOEXEC`. The fd's `f_op` is
`pidfd_fops`, supporting `poll` (POLLIN on exit), `pidfd_send_signal(2)`,
`waitid(P_PIDFD, …)`.

REQ-19: `CLONE_PARENT_SETTID` writes the child's pid in the **parent's**
PID namespace (i.e. caller's pid_ns), not the child's namespace. Important
when `CLONE_NEWPID` is also set.

REQ-20: Audit trail: a successful `clone` MUST emit `AUDIT_SYSCALL` with the
flags, pid, and namespace-IDs created (one record per ns).

## Acceptance Criteria

- [ ] AC-1: `clone(SIGCHLD, NULL, NULL, NULL, 0)` from a non-thread-group-leader child returns the new pid in the parent and 0 in the child.
- [ ] AC-2: `clone(CLONE_THREAD | CLONE_VM | CLONE_SIGHAND, …)` with `CSIGNAL=0` returns new TID; both threads share the same `tgid`.
- [ ] AC-3: `clone(CLONE_THREAD | CLONE_VM | CLONE_SIGHAND | SIGCHLD, …)` → `-EINVAL` (CSIGNAL != 0 with `CLONE_THREAD`).
- [ ] AC-4: `clone(CLONE_THREAD, …)` without `CLONE_SIGHAND` → `-EINVAL`.
- [ ] AC-5: `clone(CLONE_SIGHAND, …)` without `CLONE_VM` → `-EINVAL`.
- [ ] AC-6: `clone(CLONE_NEWUSER | CLONE_FS, …)` → `-EINVAL`.
- [ ] AC-7: `clone(CLONE_NEWPID | CLONE_THREAD, …)` → `-EINVAL`.
- [ ] AC-8: `clone(CLONE_PIDFD | CLONE_THREAD, …)` → `-EINVAL` (clone3-only combination).
- [ ] AC-9: `clone(CLONE_PIDFD, …)` writes pidfd into `*parent_tidptr`; fd is `O_CLOEXEC`.
- [ ] AC-10: `clone(0x80 | SIGCHLD, …)` (CLONE_NEWTIME via clone(2)) → `-EINVAL`.
- [ ] AC-11: `clone(SIGCHLD, NULL, &ptid, NULL, 0)` with bogus `ptid` → `-EFAULT` and no child created.
- [ ] AC-12: `clone(CLONE_CHILD_CLEARTID, …)`: zero is written to `child_tidptr` at thread-exit and a futex_wake(1) is issued.
- [ ] AC-13: `clone(CLONE_VFORK | CLONE_VM | SIGCHLD, …)`: parent blocks until child execve or exit.
- [ ] AC-14: `RLIMIT_NPROC` exceeded → `-EAGAIN` for non-CAP_SYS_RESOURCE caller.
- [ ] AC-15: 32-deep user-ns nesting → `-EUSERS`.
- [ ] AC-16: Unknown low-32 flag bit set → `-EINVAL`.
- [ ] AC-17: Auditd records syscall + flags + new pid + ns-ids.
- [ ] AC-18: x86_64 syscall number is 56; arm64 generic is 220.
- [ ] AC-19: i386 arg ordering: tls before child_tid swap is honored.
- [ ] AC-20: Per-arch register conventions: child returns with rax=0 (x86_64), x0=0 (arm64), a0=0 (riscv).

## Architecture

```rust
// arch/x86_64/entry/syscalls.rs
#[syscall(nr = 56, abi = "sysv")]
pub fn sys_clone(
    flags: u64,
    stack: u64,
    parent_tid: u64,    // user *int
    child_tid: u64,     // user *int
    tls: u64,
) -> isize {
    let args = KernelCloneArgs {
        flags:        flags & !CSIGNAL,
        exit_signal:  (flags & CSIGNAL) as u32,
        pidfd:        if flags & CLONE_PIDFD != 0 { parent_tid } else { 0 },
        child_tid:    UserPtr::new(child_tid),
        parent_tid:   UserPtr::new(parent_tid),
        stack:        UserPtr::new(stack),
        stack_size:   0,            // legacy: implicit
        tls,
        set_tid:      UserPtr::null(),
        set_tid_size: 0,
        cgroup:       0,
    };
    Fork::kernel_clone(args)
}
```

`Fork::kernel_clone(args) -> isize`:
1. /* Validate flag combinations */
2. Fork::validate_flags(&args)?  // REQ-2..REQ-10
3. /* Validate user pointers */
4. if args.flags & CLONE_PARENT_SETTID { user_access::verify_writable(args.parent_tid, 4)?; }
5. if args.flags & CLONE_PIDFD          { user_access::verify_writable(args.pidfd_uptr, 4)?; }
6. /* Allocate task_struct + thread stack */
7. let child = Fork::copy_process(&args)?;
8. /* Reserve pid */
9. let pid = PidNamespace::alloc_pid(&child, &args)?;
10. /* CLONE_PIDFD: install fd into parent's fd table */
11. let pidfd = if args.flags & CLONE_PIDFD { Pidfd::install(child, O_CLOEXEC)? } else { -1 };
12. /* CLONE_PARENT_SETTID write */
13. if args.flags & CLONE_PARENT_SETTID { user_access::put_user(pid, args.parent_tid)?; }
14. /* CLONE_PIDFD write */
15. if args.flags & CLONE_PIDFD { user_access::put_user(pidfd, args.parent_tid)?; }
16. /* Wake the child */
17. Fork::wake_up_new_task(child).
18. /* CLONE_VFORK: block on vfork_done completion */
19. if args.flags & CLONE_VFORK { wait_for_completion_killable(&child.vfork_done); }
20. return pid;

`Fork::validate_flags(args) -> Result<()>`:
1. Reject if `flags & ~KNOWN_CLONE_BITS != 0` (REQ-9).
2. Reject if `CSIGNAL` byte >= _NSIG (REQ-10).
3. Reject if `CLONE_NEWTIME` bit set via legacy clone(2) (REQ-10).
4. Reject if `CLONE_THREAD && !CLONE_SIGHAND` (REQ-2).
5. Reject if `CLONE_SIGHAND && !CLONE_VM` (REQ-2).
6. Reject if `CLONE_THREAD && CSIGNAL != 0` (REQ-3).
7. Reject if `CLONE_NEWUSER && CLONE_FS` (REQ-4).
8. Reject if `CLONE_NEWPID && CLONE_THREAD` (REQ-5).
9. Reject if `CLONE_PIDFD && CLONE_THREAD` (REQ-6 — clone(2)-only).
10. Reject if `CLONE_PIDFD && CLONE_DETACHED` (REQ-7).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_validation_total` | TOTAL | All `flags: u64` either return `-EINVAL` or pass into `copy_process`. |
| `pidfd_writes_user_only` | INVARIANT | `CLONE_PIDFD` writes only to verified user `parent_tid`. |
| `task_allocated_on_success` | INVARIANT | Return >= 0 ⟹ `task_struct` exists and is reachable from pid_ns. |
| `no_partial_task_on_failure` | INVARIANT | Return < 0 ⟹ no task_struct, no pidfd, no audit record. |
| `csignal_byte_in_range` | INVARIANT | Pre-copy_process: `CSIGNAL` byte < `_NSIG`. |
| `clone_newtime_blocked_on_clone2` | INVARIANT | `clone(2)` entry: `flags & CLONE_NEWTIME ⟹ -EINVAL`. |

### Layer 2: TLA+

`kernel/clone.tla` models the parent/child handshake:
- Per-flag-validate → per-copy_process → per-pid-alloc → per-wake → per-vfork-wait.
- Properties:
  - `safety_no_thread_with_csignal` — `CLONE_THREAD ⟹ CSIGNAL = 0`.
  - `safety_no_thread_in_new_pidns` — `CLONE_THREAD ⟹ ¬CLONE_NEWPID`.
  - `safety_pidfd_iff_requested` — pidfd installed ⟺ `CLONE_PIDFD` set.
  - `liveness_vfork_completes` — `CLONE_VFORK` ⟹ eventually parent wakes (on child exec or exit).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `kernel_clone` post: ret >= 0 ⟺ pid allocated & reachable | `Fork::kernel_clone` |
| `validate_flags` total: domain = u64, codomain = Ok ∪ {-EINVAL} | `Fork::validate_flags` |
| `pidfd install` post: cloexec set, refcount = 1 | `Pidfd::install` |
| `CLONE_CHILD_CLEARTID` registered ⟺ exit emits futex_wake | `Task::exit_path` |

### Layer 4: Verus / Creusot functional

`Per-clone semantic equivalence` to `clone(2)` man page: every flag's listed
behavior is realized exactly, including the i386 arg-swap and the x86_64
canonical order. Tested by LTP `clone01..clone09`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`clone(2)` reinforcement:

- **Per-flag sealed validation** — defense against future-bit-leak (mainline-style "unknown bits ignored" is rejected).
- **Per-user pointer pre-validation** — defense against partial-clone-then-`-EFAULT` (where pid was leaked into parent_tid before task allocation failed).
- **Per-RLIMIT_NPROC strict** — defense against per-fork-bomb.
- **Per-MAX_USER_NS_DEPTH = 32** — defense against per-userns-stack-blowup.
- **Per-CSIGNAL range-checked** — defense against per-bogus-signo invariant break.
- **Per-CLONE_NEWUSER + CLONE_FS rejection** — defense against per-cross-userns-root leak.
- **Per-pidfd `O_CLOEXEC` mandatory** — defense against per-pidfd-leak across exec.
- **Per-CLONE_THREAD + CLONE_NEWPID rejected** — defense against per-thread-in-new-pidns confusion.
- **Per-audit record on every clone** — defense against per-namespace-create elision.

## Grsecurity / PaX surface

- **PAX_RANDKSTACK at syscall entry** — randomize kernel-stack offset within the
  syscall frame so the child's kernel stack starts at a per-clone-randomized
  offset; defeats cross-syscall stack-layout reuse for ROP.
- **PaX UDEREF on user buffers** — `parent_tidptr`, `child_tidptr`, pidfd-write
  target are accessed via UDEREF-toggled segments / PAN; a kernel bug that
  forgets to validate the pointer cannot silently dereference into user space.
- **GRKERNSEC_HARDEN_PTRACE** — `CLONE_PTRACE` does NOT inherit if the parent is
  ptracing across a credential transition; combined with `PTRACE_TRACEME`
  no-yama-bypass.
- **GRKERNSEC_PROC restrictions** — newly allocated pid is published into
  `/proc/<pid>` only after the per-uid/per-group hide-filter is applied; an
  unprivileged caller cannot enumerate other users' children even after a
  successful `clone`.
- **GRKERNSEC_CHROOT** — when caller is in a chroot, deny `CLONE_NEWUSER`,
  `CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_NEWNS` (per-chroot lockdown).
- **GRKERNSEC_BRUTE** — `clone()` returning `-EAGAIN` due to `RLIMIT_NPROC` is
  counted into the per-uid forkbomb-detector; sustained over threshold
  triggers per-uid lockout.
- **GRKERNSEC_SIGNALS spoof prevention** — `CSIGNAL` value is recorded on
  child; the eventual exit-signal queues to parent with `si_code = CLD_EXITED`
  and a kernel-credentialed source, never an attacker-spoofable `SI_USER`.
- **Per-grsec `chroot_deny_pivot_root`** — interaction with `CLONE_NEWNS` from
  inside a chroot is closed: even after the namespace is created, the new
  mount-ns inherits the chroot lockdown.
- **Per-grsec `tpe` (trusted path execution)** — does not gate `clone`, but is
  evaluated at the subsequent `execve` in the child; documented here for the
  cross-syscall chain.

## Open Questions

- (none at this Tier-5 level — `clone3(2)` is the forward-compatible doorway)

## Out of Scope

- `kernel_clone` / `copy_process` task duplication (Tier-3 `kernel/fork.c`).
- `clone3(2)` syscall (separate Tier-5 doc).
- Per-arch syscall-entry shim layout (Tier-3 `arch/<arch>/entry`).
- Per-namespace internals (Tier-3 `kernel/nsproxy.c`, `ipc/namespace.c`, ...).
- Implementation code.
