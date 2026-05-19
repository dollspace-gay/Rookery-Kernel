# Tier-5 syscall: vfork(2) — syscall 58

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/fork.c (sys_vfork, SYSCALL_DEFINE0(vfork))
  - include/linux/syscalls.h (asmlinkage long sys_vfork(void))
  - arch/x86/entry/syscalls/syscall_64.tbl (58  common  vfork)
  - arch/arm64 has NO vfork — libc must use clone(CLONE_VM|CLONE_VFORK|SIGCHLD, ...)
-->

## Summary

`vfork(2)` is the suspend-the-parent-until-child-execs variant of fork. It
creates a child task that **shares the parent's address space** (`CLONE_VM`)
and **suspends the parent** (`CLONE_VFORK`) until the child either calls
`execve(2)` (in which case the child's mm is replaced and the parent is
released) or `_exit(2)` (in which case the child mm is freed and the parent
is released). The child MUST NOT modify any data, return from the current
function, or call any function other than `execve` or `_exit` — doing so is
undefined behavior because it scribbles into the parent's live stack and
heap.

On Linux `vfork` is strictly a thin shim over `kernel_clone` with the flag
combination `CLONE_VM | CLONE_VFORK | SIGCHLD`. Its sole purpose is to
avoid the COW page-table-copy cost of `fork(2)` in the common
"`fork`-then-`execve`" pattern — a 3-10x speedup for short-lived children
(shells, `system(3)`, `posix_spawn`).

`posix_spawn(3)` is the glibc forward-compatible wrapper; on modern glibc
it uses `clone(CLONE_VM | CLONE_VFORK | SIGCHLD)` directly via the
syscall, skipping the libc `vfork()` wrapper, because some libc cleanup
paths historically broke vfork's "no stack writes" rule.

This Tier-5 covers `kernel/fork.c::SYSCALL_DEFINE0(vfork, void)` (~10 lines).

## Signature

```c
long vfork(void);
```

No arguments. Returns twice — once in the parent (with child pid) after
the child has called `execve` or `_exit`, once in the child (with 0)
immediately.

## Parameters

(none)

## Return value

| Value | Meaning |
|---|---|
| `> 0` (in parent) | Child PID; returned AFTER child has called `execve` or `_exit`. |
| `0`   (in child)  | Successful vfork — child execution begins immediately; parent is suspended. |
| `-1` + `errno`    | Failure (no child created); see Errors. |

## Errors

| errno | Trigger |
|---|---|
| `EAGAIN` | `RLIMIT_NPROC` exceeded. |
| `ENOMEM` | task allocation failed. |
| `ENOSYS` | Architecture does not implement `__NR_vfork` (libc emulates via `clone`). |
| `ERESTARTNOINTR` | Signal received during pre-copy phase. |

`vfork(2)` does NOT return `-EINVAL`: no caller-supplied arguments.

## ABI surface

```text
__NR_vfork (x86_64)  = 58
__NR_vfork (i386)    = 190
__NR_vfork (powerpc) = 189
__NR_vfork (s390x)   = 190   (compat-only)
__NR_vfork (sparc)   = 85
__NR_vfork (mips O32)= 4190
__NR_vfork (mips N64)= 5113
__NR_vfork (alpha)   = 66
__NR_vfork (parisc)  = 190
__NR_vfork (arm64)   = NOT WIRED
__NR_vfork (riscv)   = NOT WIRED
__NR_vfork (loongarch) = NOT WIRED
```

Effective flag set:

```text
vfork() ≡ clone(CLONE_VM | CLONE_VFORK | SIGCHLD, NULL, NULL, NULL, 0)
       ≡ kernel_clone(KernelCloneArgs {
             flags: CLONE_VM | CLONE_VFORK,
             exit_signal: SIGCHLD,
             stack: NULL,
             ...
         })
```

## Compatibility contract

REQ-1: The syscall number is **58** on x86_64. Per-arch numbers are
architecture-stable. Where wired, removal is an ABI break.

REQ-2: On generic-syscall archs (arm64, riscv, loongarch) `__NR_vfork` is
**unwired**. Libc emulates via `clone(CLONE_VM|CLONE_VFORK|SIGCHLD, ...)`.
A direct `syscall(__NR_vfork)` on those archs returns `-ENOSYS`.

REQ-3: On archs where wired, the syscall MUST behave exactly as
`clone(CLONE_VM | CLONE_VFORK | SIGCHLD, NULL, NULL, NULL, 0)`.

REQ-4: Parent suspension: between `sys_vfork` return-in-child and child's
`execve`/`_exit`, the parent is blocked on `mm->vfork_done` completion.
Wakeup is via `complete_vfork_done()` invoked from:
- `mm_release()` (child's `exit_mm`/`exec`-time mm release).
- `exec_mmap()` (child's `execve` replaces mm).

REQ-5: Child task properties:
- **Shares parent mm** via `CLONE_VM`: page-table is the SAME `mm_struct`;
  page-table reference count is bumped.
- Child gets the parent's SP as its initial SP — meaning child runs ON the
  parent's stack until `execve`. Therefore the child MUST NOT write to the
  stack beyond what `execve`'s setup will overwrite, MUST NOT return from
  the calling function, MUST NOT call setjmp/longjmp.
- New `files_struct` cloned from caller (NOT shared — `CLONE_FILES` is 0).
- New `fs_struct` cloned.
- New `sighand_struct` cloned (NOT shared).
- New `signal_struct`.
- Same uid / gid / capabilities / no_new_privs / seccomp filters.

REQ-6: Parent's wakeup: parent's syscall returns the child's pid AFTER the
child has released the shared mm (via `execve` or `_exit`). Parent observes
the child's new mm (if execve) or zombie state (if _exit).

REQ-7: Suspend is **uninterruptible-by-signals** (`wait_for_completion_killable`
in modern kernels — interruptible only by SIGKILL, not by SIGTERM/SIGINT).
A SIGKILL during suspend kills the PARENT; the child continues
independently with the shared mm until it execve's or exits.

REQ-8: Exit notification: child's exit queues `SIGCHLD` to parent (same as
`fork`).

REQ-9: `RLIMIT_NPROC` enforced; bypass by `CAP_SYS_ADMIN`/`CAP_SYS_RESOURCE`.

REQ-10: ptrace: a vfork'd child stops at the vfork-done event if parent is
ptracing with `PTRACE_O_TRACEVFORK`; the tracer is notified before the
parent's wakeup.

REQ-11: ptrace event `PTRACE_EVENT_VFORK_DONE` is delivered to the tracer
when the parent is about to resume (after child execve/exit).

REQ-12: `vfork` MUST NOT be combined with any namespace creation (those
require a fresh mm). Since this syscall takes no args, this is automatic.

REQ-13: Audit: a successful `vfork` emits `AUDIT_SYSCALL` with the
new pid.

REQ-14: vfork-while-vfork: if the parent of a vfork is itself a vfork-
suspended task, the inner vfork is permitted; suspensions stack
(rare but legal).

## Acceptance Criteria

- [ ] AC-1: `vfork()` returns child pid in parent AFTER child execve/_exit, 0 in child immediately.
- [ ] AC-2: On x86_64, syscall number is 58.
- [ ] AC-3: On arm64, direct `syscall(__NR_vfork)` returns `-ENOSYS`; libc `vfork()` succeeds via clone.
- [ ] AC-4: Child shares parent's address space (write in child visible to parent — BUT child MUST NOT write).
- [ ] AC-5: Child has independent fd table after parent dup/close.
- [ ] AC-6: Parent is suspended; `ps` shows parent in state `S` (sleeping) or `D` until child's execve/_exit.
- [ ] AC-7: SIGKILL to parent during suspension: parent dies; child continues with shared mm.
- [ ] AC-8: SIGTERM to parent during suspension: ignored (uninterruptible by non-fatal).
- [ ] AC-9: Child exits → parent receives `SIGCHLD` with `si_code=CLD_EXITED`.
- [ ] AC-10: `RLIMIT_NPROC` exceeded → `-EAGAIN`.
- [ ] AC-11: ptracer with `PTRACE_O_TRACEVFORK`: receives vfork_done event before parent resumes.
- [ ] AC-12: Child performs `execve("/bin/true")`: parent wakes after child mm replaced.
- [ ] AC-13: Child performs `_exit(42)`: parent wakes, observes child as zombie with status 42.
- [ ] AC-14: Auditd records new pid.
- [ ] AC-15: Time-to-fork (vfork-then-execve) is ≥ 3x faster than fork-then-execve under load (perf-regression guard).

## Architecture

```rust
#[syscall(nr = 58, abi = "sysv", arches = "x86_64,i386,powerpc,sparc,mips,s390x,alpha,parisc,m68k,sh,xtensa,microblaze")]
pub fn sys_vfork() -> isize {
    let args = KernelCloneArgs {
        flags:        CLONE_VFORK | CLONE_VM,
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

`Fork::kernel_clone(args)` with `CLONE_VFORK`:
1. Standard copy_process path; child shares mm via `CLONE_VM`.
2. Allocate `child.vfork_done = Completion::new()`.
3. Wake_up_new_task(child).
4. /* Parent suspends */
5. wait_for_completion_killable(&child.vfork_done).
6. /* Wakeup paths */
7. ... (in child's `execve` / `exit_mm`): complete_vfork_done(&parent.vfork_done).
8. Return child pid.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vfork_flag_set` | INVARIANT | `sys_vfork`: KernelCloneArgs.flags == CLONE_VM | CLONE_VFORK; exit_signal == SIGCHLD. |
| `vfork_completion_pairs` | INVARIANT | Every vfork() with CLONE_VFORK ⟹ exactly one complete_vfork_done() in child path. |
| `parent_resume_after_release` | INVARIANT | Parent does not resume userspace before child's mm_release / exec_mmap. |

### Layer 2: TLA+

`kernel/vfork.tla`:
- States: PARENT_WAIT → CHILD_RUN → CHILD_EXECVE/EXIT → COMPLETE → PARENT_RESUME.
- Properties:
  - `safety_parent_blocked_until_release` — parent's syscall return-pc not reached while child holds shared mm.
  - `safety_sigkill_unblocks_parent` — SIGKILL to parent ⟹ parent exits even if child holds mm.
  - `liveness_eventual_release` — child eventually execve's or exits ⟹ parent resumes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_vfork` post: args literal-equivalent to clone(CLONE_VM|CLONE_VFORK|SIGCHLD) | `sys_vfork` |
| `vfork_done` completion: initialized in copy_process, completed in mm_release | `Fork::copy_process`, `Mm::mm_release` |
| Parent's user-RIP unchanged across vfork_done wait | scheduler entry |

### Layer 4: Verus / Creusot functional

POSIX `vfork(2)` semantic equivalence. LTP `vfork01..vfork02` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`vfork(2)` reinforcement:

- **Per-CLONE_VM strict mm-share** — defense against per-page-table-double-free (refcount enforced).
- **Per-CLONE_VFORK completion idempotent** — defense against per-double-complete (use-after-free of parent's `vfork_done`).
- **Per-parent suspend `_killable`** — defense against per-vfork-deadlock (SIGKILL can always unblock).
- **Per-`RLIMIT_NPROC` strict** — defense against per-fork-bomb.
- **Per-ptrace vfork-done event delivered before parent resume** — defense against per-tracer-miss.
- **Per-arch absence on generic-sys archs** — defense against per-legacy-ABI surface bloat.
- **Per-audit record on success** — defense against per-process-creation elision.
- **Per-`SIGCHLD` queued from kernel-source** — defense against per-spoofed-exit-notification.

## Grsecurity / PaX surface

- **PAX_RANDKSTACK at syscall entry** — `sys_vfork` randomizes kernel stack
  offset; combined with the shared-mm setup, the child's first kernel-stack
  frame is at a per-vfork-randomized address.
- **PaX UDEREF** — `sys_vfork` has no user pointers; subsequent
  child-side `execve` uses UDEREF/PAN-toggled access for the path/argv
  buffers.
- **GRKERNSEC_HARDEN_PTRACE** — `PTRACE_O_TRACEVFORK` is gated by the
  yama+grsec policy; cross-credential ptrace inherit is denied.
- **GRKERNSEC_PROC restrictions** — newly forked pid published into
  `/proc/<pid>` only after per-uid hide-filter applied.
- **GRKERNSEC_CHROOT** — `vfork(2)` permitted inside chroot; child inherits
  chroot lockdown. The immediately-following `execve` is gated by
  per-grsec chroot_findtask / trusted-path-execution rules.
- **GRKERNSEC_BRUTE** — vfork() returning `-EAGAIN` counts into the per-uid
  forkbomb detector.
- **GRKERNSEC_SIGNALS spoof prevention** — `SIGCHLD` on child exit is
  kernel-sourced; `si_code = CLD_EXITED`/`CLD_KILLED`; un-spoofable.
- **Per-grsec `chroot_pivot_root`** — child running in chroot cannot
  pivot_root even with shared mm; the share gives access to parent's vmas
  but not to a chroot-escape primitive.
- **Per-grsec `dmesg` restriction** — vfork-rate spikes logged at most
  once per second; readable only by `CAP_SYSLOG`.

## Open Questions

- (none at this Tier-5 level — vfork is a closed legacy primitive)

## Out of Scope

- `clone(2)` / `clone3(2)` / `fork(2)` syscalls (separate Tier-5 docs).
- `execve(2)` mm replacement (separate Tier-5 doc).
- `kernel_clone` / `copy_process` task duplication (Tier-3 `kernel/fork.c`).
- ptrace event delivery (Tier-3 `kernel/ptrace.c`).
- Implementation code.
