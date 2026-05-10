# Tier-3: kernel/task-lifecycle — fork, exit, exec, signal, kthread, cred, capability, pid, namespaces

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - kernel/fork.c
  - kernel/exit.c
  - kernel/signal.c
  - kernel/exec_domain.c
  - kernel/kthread.c
  - kernel/cred.c
  - kernel/groups.c
  - kernel/capability.c
  - kernel/user.c
  - kernel/user_namespace.c
  - kernel/pid.c
  - kernel/pid_namespace.c
  - kernel/nsproxy.c
  - kernel/ucount.c
  - kernel/rseq.c
  - kernel/sys.c
  - include/linux/sched.h
  - include/linux/sched/task.h
  - include/linux/cred.h
  - include/linux/pid.h
  - include/linux/pid_namespace.h
  - include/linux/nsproxy.h
  - include/linux/signal.h
  - include/linux/capability.h
  - include/uapi/linux/capability.h
-->

## Summary
Tier-3 design for the per-task lifecycle: process creation (fork/clone/clone3), termination (exit/exit_group), signal delivery + handling, kernel-thread management (kthread_create/run/stop), credentials (struct cred), capabilities, PID + PID-namespace, nsproxy (namespace bundle pointer), user namespaces, ucount accounting, restartable sequences (rseq). Owns the **`PR_REQUEST_EXEC_GAIN` prctl** that, with `00-security-principles.md`'s ELF-note recognition (in `fs/exec-binfmt.md`), sets per-task `exec_gain_state`.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Process creation (fork/clone/clone3) | `kernel/fork.c` |
| Process termination | `kernel/exit.c` |
| Signal delivery + handling | `kernel/signal.c`, `include/linux/signal.h` |
| Personality / exec_domain | `kernel/exec_domain.c` |
| Kernel-thread mgmt | `kernel/kthread.c` |
| Credentials (struct cred) | `kernel/cred.c`, `include/linux/cred.h` |
| Groups (supplementary GIDs) | `kernel/groups.c` |
| Capabilities (POSIX) | `kernel/capability.c`, `include/linux/capability.h`, `include/uapi/linux/capability.h` |
| User accounting | `kernel/user.c` |
| User namespaces | `kernel/user_namespace.c` |
| PID + PID namespace | `kernel/pid.c`, `kernel/pid_namespace.c`, `include/linux/pid.h`, `include/linux/pid_namespace.h` |
| nsproxy | `kernel/nsproxy.c`, `include/linux/nsproxy.h` |
| ucount per-namespace counters | `kernel/ucount.c` |
| rseq (restartable sequences) | `kernel/rseq.c` |
| Misc syscalls (rlimit, prlimit64, setpgid, getpid, prctl, gettid, ...) | `kernel/sys.c` |
| Top-level task structure | `include/linux/sched.h`, `include/linux/sched/task.h` |

## Compatibility contract

### Syscall surface

`fork`, `vfork`, `clone`, `clone3`, `execve`, `execveat`, `exit`, `exit_group`, `wait4`, `waitid`, `setpgid`, `getpgid`, `setsid`, `getsid`, `setrlimit`, `getrlimit`, `prlimit64`, `prctl`, `arch_prctl`, `set_tid_address`, `gettid`, `getpid`, `getppid`, `getpgrp`, `setns`, `unshare`, `kill`, `tkill`, `tgkill`, `pidfd_open`, `pidfd_send_signal`, `rt_sigaction`, `rt_sigprocmask`, `rt_sigpending`, `rt_sigqueueinfo`, `rt_sigsuspend`, `rt_sigreturn`, `rt_sigtimedwait`, `sigaltstack`, `signalfd`, `signalfd4`, `pause`, `setuid`, `seteuid`, `setresuid`, `setreuid`, `getuid`, `geteuid`, `getresuid`, `setgid`, `setegid`, `setresgid`, `setregid`, `getgid`, `getegid`, `getresgid`, `setgroups`, `getgroups`, `setfsuid`, `setfsgid`, `capset`, `capget`, `personality`, `umask`, `times`, `getrusage`, `getcpu`, `pidfd_getfd`.

(Each gets a Tier-5 `uapi/syscalls/<name>.md` in Phase D.)

### Capability bits

`include/uapi/linux/capability.h` enumerates 41 capability bits at baseline (CAP_CHOWN=0 through CAP_CHECKPOINT_RESTORE=40). Numeric values + per-cap permission semantics byte-identical.

### `task_struct` layout

`include/linux/sched.h` defines `struct task_struct` (~700 bytes, ~200 fields). First-cache-line + commonly-macro-accessed fields layout-equivalent to upstream:
- `state` (now split into `__state` + `running`)
- `stack` pointer
- `flags`
- `pid`, `tgid`
- `prio`, `static_prio`, `normal_prio`, `rt_priority`
- `sched_class`, `se`, `rt`, `dl` (per-class scheduling state)
- `mm`, `active_mm` (cross-ref `mm/00-overview.md`)
- `files`, `fs`, `nsproxy`, `cred`, `signal`, `sighand` (refcounted bundles)
- Rookery-NEW: `exec_gain_state` byte (cross-ref `00-security-principles.md`)

### `/proc/<pid>/*` content fields

Owned partly here, partly by other subsystems:
- `/proc/<pid>/stat` — task state, name, scheduler params, mm sizes (cross-ref `mm/`), VM counters
- `/proc/<pid>/status` — name, State, Tgid, Ngid, Pid, PPid, TracerPid, Uid, Gid, FDSize, Groups, NStgid, NSpid, NSpgid, NSsid, VmPeak, VmSize, VmLck, ..., SigQ, SigPnd, SigBlk, SigIgn, SigCgt, CapInh, CapPrm, CapEff, CapBnd, CapAmb, NoNewPrivs, Seccomp, Speculation_Store_Bypass, Cpus_allowed, Cpus_allowed_list, Mems_allowed, Mems_allowed_list, voluntary_ctxt_switches, nonvoluntary_ctxt_switches
- `/proc/<pid>/comm` — task name
- `/proc/<pid>/limits` — rlimit values
- `/proc/<pid>/personality` — personality bits
- `/proc/<pid>/ns/*` — namespace symlinks (mnt, ipc, net, uts, pid, user, cgroup, time)
- `/proc/<pid>/uid_map`, `/proc/<pid>/gid_map`, `/proc/<pid>/projid_map` — userns mapping
- `/proc/<pid>/setgroups` — userns setgroups policy

All format-identical.

### Signal numbers + flags

POSIX 1..64; signal flags `SA_NOCLDSTOP`, `SA_NOCLDWAIT`, `SA_SIGINFO`, `SA_RESTORER`, `SA_ONSTACK`, `SA_RESTART`, `SA_NODEFER`, `SA_RESETHAND`, `SA_UNSUPPORTED`, `SA_EXPOSE_TAGBITS`. Identical numeric values + semantics.

### Exec-gain prctl

**`PR_REQUEST_EXEC_GAIN` = 80** (per `00-security-principles.md` knob inventory — subject to upstream prctl-number assignment confirmation at implementation time).

```c
int prctl(PR_REQUEST_EXEC_GAIN, unsigned long flags, 0, 0, 0);
```

`flags` is the bit-set mirroring `NT_ROOKERY_SECURITY_FLAGS` ELF note's `n_desc`:
- bit 0: `ROOKERY_SECURITY_NEEDS_EXEC_GAIN` — allows W→X transitions in mprotect
- bit 1: `ROOKERY_SECURITY_NEEDS_RWX_ANON` — allows mmap with PROT_WRITE|PROT_EXEC on anon

Returns 0 on success, `-EPERM` if NoNewPrivs is set, `-EBUSY` if any executable mapping already exists, `-EINVAL` for unknown bits.

Once granted, sticky for task lifetime. `clone(2)` inherits state; `execve(2)` resets state (re-evaluated against new ELF note).

## Requirements

- REQ-1: Every task-lifecycle syscall has byte-identical entry/exit ABI per upstream.
- REQ-2: `clone` flag set + `clone3 cl_args` struct layout match upstream byte-for-byte; CLONE_*-flag semantics identical.
- REQ-3: `task_struct` first-cache-line + commonly-accessed-via-macro fields layout-equivalent; `pahole struct task_struct` of those fields produces identical offsets.
- REQ-4: `exit_state` transitions (TASK_RUNNING → ... → TASK_DEAD) match upstream; ptrace stop semantics preserved.
- REQ-5: Signal queue, signal mask, alternate-stack semantics match upstream byte-for-byte. POSIX queue limits enforced.
- REQ-6: Signal delivery during system call: restart vs. abort decision per `SA_RESTART` matches upstream.
- REQ-7: kthread mgmt: `kthread_create`, `kthread_run`, `kthread_stop` semantics identical. New abstraction `kernel::task::Kthread` per issue #4.
- REQ-8: Cred: `prepare_creds` / `commit_creds` / `revert_creds` semantics identical; creds RCU-managed. `current_cred()` macro layout-identical.
- REQ-9: Capabilities (POSIX file caps + ambient set + bounding set + permitted + effective + inherited) match upstream semantics. `capset` / `capget` byte-identical.
- REQ-10: PID allocation via per-namespace IDR identical; PID 0 reserved; PID 1 (init) per-namespace.
- REQ-11: PID namespace creation via `unshare(CLONE_NEWPID)` + `setns` semantics identical; child task observes its in-namespace PID.
- REQ-12: User namespace mapping via `/proc/<pid>/uid_map` write semantics identical; `setgroups` writeability gated per `/proc/<pid>/setgroups` policy.
- REQ-13: nsproxy refcounting + namespace bundle (mnt/ipc/net/uts/pid/cgroup/time/user) atomic switching match upstream.
- REQ-14: ucount per-namespace counters (RLIMIT_NOFILE, _SIGPENDING, _MSGQUEUE, _MEMLOCK, _NPROC) per upstream + per `/proc/sys/user/`.
- REQ-15: rseq syscall + per-task rseq area + sequence-counter abort semantics match upstream.
- REQ-16: `prctl` + `arch_prctl` opcode set: every documented PR_*/ARCH_* opcode preserved; **PR_REQUEST_EXEC_GAIN=80 added as Rookery extension** per `00-security-principles.md`. Inherits across `clone(2)`; resets across `execve(2)` (re-evaluated against new ELF note).
- REQ-17: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `strace` golden trace of fork/exec/exit/wait sequence byte-identical vs. upstream. (covers REQ-1)
- [ ] AC-2: `clone3` selftest (`tools/testing/selftests/clone3/`) passes with same set as upstream. (covers REQ-2)
- [ ] AC-3: `pahole struct task_struct | head -30` shows byte-identical first-cache-line layout vs. upstream. (covers REQ-3)
- [ ] AC-4: ptrace selftest (`tools/testing/selftests/ptrace/`) passes. (covers REQ-4)
- [ ] AC-5: Signal selftests (`tools/testing/selftests/signals/`) pass. (covers REQ-5, REQ-6)
- [ ] AC-6: A kthread test creates 100 kthreads with `kthread_run`; each runs to completion; `kthread_stop` returns 0 each. (covers REQ-7)
- [ ] AC-7: Capability selftest (`tools/testing/selftests/capabilities/`) passes. (covers REQ-8, REQ-9)
- [ ] AC-8: PID/namespace selftests (`tools/testing/selftests/pid_namespace/`, `pidfd/`) pass. (covers REQ-10, REQ-11)
- [ ] AC-9: User-namespace selftest (`tools/testing/selftests/user_namespace/`) passes. (covers REQ-12)
- [ ] AC-10: An `unshare -mUipnf` runs an inner shell with all namespaces switched; nsproxy refcounts observable via `/proc/<pid>/ns/*` symlinks. (covers REQ-13)
- [ ] AC-11: `prlimit64`/`getrlimit`/`setrlimit` honor ucount per-namespace limits. (covers REQ-14)
- [ ] AC-12: rseq selftest (`tools/testing/selftests/rseq/`) passes. (covers REQ-15)
- [ ] AC-13: A test calls `prctl(PR_REQUEST_EXEC_GAIN, 1, 0, 0, 0)`; returns 0; subsequent `mprotect(PROT_READ|PROT_WRITE|PROT_EXEC, ...)` succeeds. Without prctl + without ELF note, mprotect returns EACCES. After `execve` to a binary without the note, exec_gain_state resets. (covers REQ-16)
- [ ] AC-14: Hardening section present and follows template. (covers REQ-17)

## Architecture

### Rust module organization

- `kernel::task::Task` — `task_struct` wrapper (cross-ref existing rust-for-linux `rust/kernel/task.rs`)
- `kernel::task::fork` — fork/clone/clone3
- `kernel::task::exit` — exit/exit_group + reaping
- `kernel::task::signal` — signal queue + delivery + handling
- `kernel::task::kthread::Kthread` — kthread spawn (NEW abstraction; issue #4)
- `kernel::cred::Cred` — credentials (cross-ref existing `rust/kernel/cred.rs`)
- `kernel::cred::cap::Capability` — capability set
- `kernel::pid::Pid`, `kernel::pid::PidNs` — PID + namespace
- `kernel::userns::UserNs` — user namespace + uid_map / gid_map
- `kernel::nsproxy::NsProxy` — namespace bundle
- `kernel::ucount::Ucount` — per-namespace counters
- `kernel::rseq::RseqArea` — rseq area + abort logic
- `kernel::task::prctl` — prctl handler dispatcher (incl. PR_REQUEST_EXEC_GAIN)
- `kernel::task::exec_gain` — per-task exec_gain_state management

### Locking and concurrency

- **`tasklist_lock`**: rwlock. Read-side: walking task list. Write-side: fork/exit list manipulation.
- **`task_struct->alloc_lock`**: spinlock. Protects `cred`, `mm`, `files`, `fs` simultaneous swap.
- **`task_struct->sighand->siglock`**: spinlock. Signal queue + mask.
- **`signal_struct` lock**: per-thread-group; protects shared signal state.
- **PID hash table**: per-namespace IDR + RCU.
- **nsproxy**: refcounted; readers use rcu_dereference; writer uses `task_lock`.
- **cred**: RCU-managed; `current_cred()` returns `&'a Cred` borrowed from current task; `prepare_creds` clones, `commit_creds` swaps under task_lock.

### Error handling

- `Err(EAGAIN)` — fork limit reached (RLIMIT_NPROC)
- `Err(ENOMEM)` — task_struct alloc failed
- `Err(EPERM)` — capability check failed; cred change requires root
- `Err(ECHILD)` — wait4: no children
- `Err(ESRCH)` — kill: no such process
- `Err(EINVAL)` — bad signal / bad flags
- `Err(EBUSY)` — prctl(PR_REQUEST_EXEC_GAIN) called after executable mapping already exists

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `task_struct` allocation + initialization (large struct; ensure no field is left uninitialized in the safety-relevant subset) | `kani::proofs::kernel::task::alloc_safety` |
| Signal queue insertion + dequeue under siglock | `kani::proofs::kernel::task::signal_queue_safety` |
| Cred swap under alloc_lock | `kani::proofs::kernel::task::cred_swap_safety` |
| nsproxy refcount + RCU read | `kani::proofs::kernel::task::nsproxy_safety` |
| PR_REQUEST_EXEC_GAIN state mutation | `kani::proofs::kernel::task::exec_gain_safety` |
| kthread spawn / stop coordination | `kani::proofs::kernel::task::kthread_safety` |

### Layer 2: TLA+ models

- `models/kernel/task/clone_exec_inheritance.tla` (NEW) — proves CLONE_VM/CLONE_FS/CLONE_FILES inheritance of cred + nsproxy + exec_gain_state preserves invariants under racing fork+execve.
- `models/kernel/task/signal_delivery.tla` (NEW) — proves signal-pending semantics: a signal sent via kill(2) is delivered exactly once to the targeted thread (or thread group leader).
- `models/kernel/task/pid_alloc.tla` (NEW) — proves PID allocation across namespaces: no two tasks in the same namespace share a PID; PID is recycled only after task is reaped.
- `models/arch/x86/exec_gain_propagation.tla` (inherited from `arch/x86/entry.md`) — co-owned. Proves per-task exec_gain_state read in mmap/mprotect is consistent with the value set here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Task list (per-namespace) | Every alive task is reachable from init_task; no cycles | `kani::proofs::kernel::task::task_list_invariants` |
| PID hash table (per-namespace IDR) | Every assigned PID maps to exactly one task; PID 0 reserved | `kani::proofs::kernel::pid::idr_invariants` |
| Signal queue | Pending count ≤ MAX_PENDING; queue non-empty iff pending count > 0 | `kani::proofs::kernel::task::signal_queue_invariants` |
| Cred RCU chain | Old creds reachable until grace period elapses; new cred is current | `kani::proofs::kernel::task::cred_rcu_invariants` |
| nsproxy refcount | Refcount > 0 iff at least one task references; reaches 0 → freed | `kani::proofs::kernel::task::nsproxy_refcount_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Capability inheritance arithmetic** via Creusot — proves: given parent's cap sets + child's exec spec, the child's cap sets follow the upstream-defined formula (the kernel's "P' = (P & inheritable) | (F & permitted)" algebra).
- **PID alloc across namespaces** via Verus — proves no double-allocation across racing fork+exit.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MPROTECT-W→X-block** (per-task state + prctl) | `PR_REQUEST_EXEC_GAIN` prctl; per-task `exec_gain_state` field; clone inherits, execve resets per ELF note | § Default-on system-wide with per-process exemption |
| **NOEXEC-strict** (same prctl/state) | Same | § Default-on system-wide with per-process exemption |
| **PR_SET_NO_NEW_PRIVS** | Existing upstream feature; preserved unchanged. Setting NoNewPrivs prevents subsequent `PR_REQUEST_EXEC_GAIN` from succeeding (returns EPERM). | (matches upstream) |
| **PR_SET_MDWE** | Existing upstream Memory-Deny-Write-Execute prctl; preserved unchanged. Orthogonal to Rookery's exec_gain — MDWE always tightens; exec_gain only loosens (and only for tasks that legitimately need to JIT). | (matches upstream) |
| **REFCOUNT** (`task_struct->usage`, `task_struct->stack_refcount`, etc.) | All task-related refcounts use `Refcount` (saturating) | § Mandatory |

### Row-1 features consumed by this component

- **PRIVATE_KSTACKS**: per-task kernel stack allocated at fork; isolated via `arch/x86/00-overview.md` § kernel-platform.md
- **AUTOSLAB**: `task_struct` allocated via `KmemCache::<TaskStruct>::new()` per-type cache
- **MEMORY_SANITIZE**: task_struct cache freed objects zeroed
- **CONSTIFY**: capability-set bit-position tables, signal-default-action tables are static const
- **SIZE_OVERFLOW**: signal-queue size accounting uses checked operators

### Row-2 / GR-RBAC integration

Numerous LSM hooks fire from this component:
- `security_task_alloc(task)` — on task creation
- `security_task_free(task)` — on task destruction
- `security_task_setpgid`, `security_task_getpgid`, etc.
- `security_cred_alloc_blank`, `security_cred_prepare`, `security_cred_transfer`
- `security_capable(cap)` — every capability check
- `security_setprocattr` / `security_getprocattr` — `/proc/<pid>/attr/*` writes
- `security_task_kill(target, info, sig, cred)` — kill(2) hook
- `security_task_prctl(option, ...)` — every prctl

GR-RBAC (per its loaded policy) can deny any of these. The default-empty policy preserves drop-in compat.

### Userspace-visible behavior changes

Per Axiom 4 of `00-security-principles.md`:
- **PR_REQUEST_EXEC_GAIN** is a NEW prctl number (80). Documented in this Tier-3 + `00-security-principles.md`. JIT runtimes need it (or the ELF note); non-JIT processes never call it.
- All other behavior matches upstream.

### Verification

(See § Verification above.)

## Open Questions

(none — task-lifecycle syscalls are exhaustively specified by POSIX + Linux extensions; no architectural ambiguities at this tier)

## Out of Scope

- Architecture-specific task state save/restore (cross-ref `arch/x86/00-overview.md` § entry.md, kernel-platform.md)
- Scheduling policy (cross-ref `kernel/sched/`)
- Cgroup integration (cross-ref `kernel/cgroup/`)
- Implementation code
