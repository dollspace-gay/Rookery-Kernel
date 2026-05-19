---
title: "Tier-5 UAPI: include/uapi/linux/sched.h — Scheduler + clone(2) ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The sched UAPI is the syscall-edge contract for two intertwined kernel surfaces: the **task-creation family** (`clone(2)`, `clone3(2)`, `fork(2)`, `vfork(2)`, `unshare(2)`, `setns(2)`) and the **scheduler-policy family** (`sched_setscheduler(2)`, `sched_setparam(2)`, `sched_getscheduler(2)`, `sched_getparam(2)`, `sched_setattr(2)`, `sched_getattr(2)`, `sched_yield(2)`, `sched_get_priority_min/max(2)`, `sched_rr_get_interval(2)`, `sched_setaffinity(2)`, `sched_getaffinity(2)`). It is consumed by every libc thread-creation primitive (`pthread_create`), every container runtime (Docker, runc, podman, systemd-nspawn, LXC), every real-time application (`SCHED_FIFO`/`SCHED_RR`), every deadline-scheduled task (`SCHED_DEADLINE`), and every cgroup-aware orchestrator. It defines:

- **Per-bit `CLONE_*` flags** in the low 32 bits, sharing the bit space with `CSIGNAL` (low 8 bits = exit signal): `CLONE_VM`, `CLONE_FS`, `CLONE_FILES`, `CLONE_SIGHAND`, `CLONE_PIDFD`, `CLONE_PTRACE`, `CLONE_VFORK`, `CLONE_PARENT`, `CLONE_THREAD`, `CLONE_NEWNS`, `CLONE_SYSVSEM`, `CLONE_SETTLS`, `CLONE_PARENT_SETTID`, `CLONE_CHILD_CLEARTID`, `CLONE_DETACHED` (unused/ignored), `CLONE_UNTRACED`, `CLONE_CHILD_SETTID`, `CLONE_NEWCGROUP`, `CLONE_NEWUTS`, `CLONE_NEWIPC`, `CLONE_NEWUSER`, `CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_IO`.
- **Per-bit clone3-only flags** in the upper 32 bits (only accessible from `clone3(2)` via a u64 flags field): `CLONE_CLEAR_SIGHAND`, `CLONE_INTO_CGROUP`, `CLONE_AUTOREAP`, `CLONE_NNP` (set `no_new_privs`), `CLONE_PIDFD_AUTOKILL` (kill child when pidfd closes), `CLONE_EMPTY_MNTNS`.
- **`CSIGNAL = 0xff`** — low 8 bits of the `clone(2)` flags argument carry the signal to send to the parent on child exit; `clone3(2)` carries the same value in `clone_args.exit_signal`.
- **`CLONE_NEWTIME = 0x80`** — sits inside the `CSIGNAL` mask, so it's usable only with `unshare(2)` and `clone3(2)` (which uses `exit_signal` as a separate u64 field).
- **`UNSHARE_EMPTY_MNTNS = 0x100000`** — `unshare`-only flag overlapping `CLONE_PARENT_SETTID` bit space.
- **`struct clone_args`** (the `clone3(2)` argument block): `{ flags, pidfd, child_tid, parent_tid, exit_signal, stack, stack_size, tls, set_tid, set_tid_size, cgroup }` all `__aligned_u64`; versioned by size via `CLONE_ARGS_SIZE_VER0 = 64`, `CLONE_ARGS_SIZE_VER1 = 80`, `CLONE_ARGS_SIZE_VER2 = 88`.
- **Scheduling policies** `SCHED_NORMAL = 0` (CFS / EEVDF), `SCHED_FIFO = 1`, `SCHED_RR = 2`, `SCHED_BATCH = 3`, `SCHED_IDLE = 5` (note the gap: 4 is reserved for unimplemented SCHED_ISO), `SCHED_DEADLINE = 6`, `SCHED_EXT = 7` (BPF scheduler class).
- **`SCHED_RESET_ON_FORK = 0x40000000`** — OR'd into the policy to clear elevated RT class on `fork(2)`.
- **`SCHED_FLAG_*` for `sched_{set,get}attr(2)`**: `SCHED_FLAG_RESET_ON_FORK = 0x01`, `SCHED_FLAG_RECLAIM = 0x02`, `SCHED_FLAG_DL_OVERRUN = 0x04`, `SCHED_FLAG_KEEP_POLICY = 0x08`, `SCHED_FLAG_KEEP_PARAMS = 0x10`, `SCHED_FLAG_UTIL_CLAMP_MIN = 0x20`, `SCHED_FLAG_UTIL_CLAMP_MAX = 0x40`, plus composites `SCHED_FLAG_KEEP_ALL` and `SCHED_FLAG_UTIL_CLAMP`, and the all-bits mask `SCHED_FLAG_ALL`.
- **`SCHED_GETATTR_FLAG_DL_DYNAMIC = 0x01`** — sched_getattr-only output flag indicating SCHED_DEADLINE bandwidth dynamic adjustment is in effect.
- **`struct sched_attr`** (the `sched_setattr(2)` / `sched_getattr(2)` arg): `{ u32 size; u32 sched_policy; u64 sched_flags; s32 sched_nice; u32 sched_priority; u64 sched_runtime; u64 sched_deadline; u64 sched_period; u32 sched_util_min; u32 sched_util_max; }`. Versioned by `SCHED_ATTR_SIZE_VER0 = 48`, `SCHED_ATTR_SIZE_VER1 = 56`.

Critical for: every container start (clone3 with `CLONE_NEW{NS,UTS,IPC,USER,PID,NET,CGROUP,TIME}` to assemble a namespace bundle), every realtime application (PREEMPT_RT loads, audio servers, robotics, automotive ECUs running SCHED_FIFO), every deadline-scheduled workload (SCHED_DEADLINE), every BPF scheduler experiment (`SCHED_EXT`), every JVM/Go/Rust runtime that uses `sched_yield`/`sched_setaffinity` for spin-tuning, every CRIU checkpoint that must replay clone-tree topology.

This Tier-5 covers `include/uapi/linux/sched.h` (~162 lines) plus `include/uapi/linux/sched/types.h` (~122 lines) — the `struct sched_attr` definition. The corresponding implementations (`kernel/fork.c`, `kernel/sched/core.c`, `kernel/sched/fair.c`, `kernel/sched/rt.c`, `kernel/sched/deadline.c`, `kernel/sched/ext.c`) are Tier-3 docs separately.

### Acceptance Criteria

- [ ] AC-1: `clone(CLONE_THREAD, ...)` without `CLONE_SIGHAND`: returns `-EINVAL`.
- [ ] AC-2: `clone(CLONE_SIGHAND, ...)` without `CLONE_VM`: returns `-EINVAL`.
- [ ] AC-3: `clone(CLONE_NEWPID | CLONE_THREAD, ...)`: returns `-EINVAL`.
- [ ] AC-4: `clone(CLONE_NEWUSER | CLONE_FS, ...)`: returns `-EINVAL`.
- [ ] AC-5: `clone3(args, size != one_of(64, 80, 88, +8N) )`: returns `-EINVAL`.
- [ ] AC-6: `clone3` with trailing nonzero bytes beyond known size: returns `-E2BIG`.
- [ ] AC-7: `clone_args.exit_signal >= 128`: returns `-EINVAL`.
- [ ] AC-8: `CLONE_PIDFD` writes a valid pidfd; `fstat(pidfd).st_mode` has `S_IFIFO` semantics (S_ISFIFO false, but is_pidfd via fcntl).
- [ ] AC-9: `CLONE_PIDFD_AUTOKILL` without `CLONE_PIDFD`: returns `-EINVAL`.
- [ ] AC-10: `CLONE_PIDFD_AUTOKILL`: closing the last pidfd ref delivers `SIGKILL` to child.
- [ ] AC-11: `CLONE_NNP`: child has `prctl(PR_GET_NO_NEW_PRIVS) == 1`.
- [ ] AC-12: `CLONE_INTO_CGROUP` with invalid cgroup fd: returns `-EBADF`.
- [ ] AC-13: `SCHED_NORMAL == 0`, `SCHED_FIFO == 1`, `SCHED_RR == 2`, `SCHED_DEADLINE == 6`, `SCHED_EXT == 7`.
- [ ] AC-14: `SCHED_ISO` (value 4): `sched_setscheduler` returns `-EINVAL`.
- [ ] AC-15: `sched_setscheduler(SCHED_FIFO, prio=50)` without `CAP_SYS_NICE`: returns `-EPERM`.
- [ ] AC-16: `sched_setscheduler(SCHED_FIFO, prio=100)`: returns `-EINVAL` (out of range).
- [ ] AC-17: `sched_setattr({size=47})`: returns `-E2BIG`.
- [ ] AC-18: `sched_setattr` unaligned: returns `-EFAULT`.
- [ ] AC-19: `SCHED_FLAG_DL_OVERRUN` with `SCHED_NORMAL`: returns `-EINVAL`.
- [ ] AC-20: `SCHED_FLAG_UTIL_CLAMP_MIN` with `util_min > 1024`: returns `-EINVAL`.
- [ ] AC-21: `SCHED_FLAG_UTIL_CLAMP_MIN` with `util_min > util_max`: returns `-EINVAL`.
- [ ] AC-22: `SCHED_RESET_ON_FORK` set on parent: child of fork has `SCHED_NORMAL`, nice 0.
- [ ] AC-23: `sizeof(struct clone_args) >= CLONE_ARGS_SIZE_VER2 == 88`.
- [ ] AC-24: `sizeof(struct sched_attr) >= SCHED_ATTR_SIZE_VER1 == 56`.
- [ ] AC-25: `CLONE_NEWTIME` via `clone(2)` (low 32 bits): returns `-EINVAL` (it overlaps CSIGNAL); via `clone3(2)`: succeeds.

### Architecture

```
// Low-32 CLONE_* bits (also usable via clone(2))
pub const CSIGNAL: u64               = 0x000000ff;
pub const CLONE_VM: u64              = 0x00000100;
pub const CLONE_FS: u64              = 0x00000200;
pub const CLONE_FILES: u64           = 0x00000400;
pub const CLONE_SIGHAND: u64         = 0x00000800;
pub const CLONE_PIDFD: u64           = 0x00001000;
pub const CLONE_PTRACE: u64          = 0x00002000;
pub const CLONE_VFORK: u64           = 0x00004000;
pub const CLONE_PARENT: u64          = 0x00008000;
pub const CLONE_THREAD: u64          = 0x00010000;
pub const CLONE_NEWNS: u64           = 0x00020000;
pub const CLONE_SYSVSEM: u64         = 0x00040000;
pub const CLONE_SETTLS: u64          = 0x00080000;
pub const CLONE_PARENT_SETTID: u64   = 0x00100000;
pub const CLONE_CHILD_CLEARTID: u64  = 0x00200000;
pub const CLONE_DETACHED: u64        = 0x00400000;
pub const CLONE_UNTRACED: u64        = 0x00800000;
pub const CLONE_CHILD_SETTID: u64    = 0x01000000;
pub const CLONE_NEWCGROUP: u64       = 0x02000000;
pub const CLONE_NEWUTS: u64          = 0x04000000;
pub const CLONE_NEWIPC: u64          = 0x08000000;
pub const CLONE_NEWUSER: u64         = 0x10000000;
pub const CLONE_NEWPID: u64          = 0x20000000;
pub const CLONE_NEWNET: u64          = 0x40000000;
pub const CLONE_IO: u64              = 0x80000000;

// CLONE_NEWTIME — inside CSIGNAL bits; clone3(2) / unshare(2) only.
pub const CLONE_NEWTIME: u64         = 0x00000080;

// unshare-only
pub const UNSHARE_EMPTY_MNTNS: u64   = 0x00100000;

// Upper-32 clone3(2)-only flags
pub const CLONE_CLEAR_SIGHAND: u64   = 1 << 32;
pub const CLONE_INTO_CGROUP: u64     = 1 << 33;
pub const CLONE_AUTOREAP: u64        = 1 << 34;
pub const CLONE_NNP: u64             = 1 << 35;
pub const CLONE_PIDFD_AUTOKILL: u64  = 1 << 36;
pub const CLONE_EMPTY_MNTNS: u64     = 1 << 37;

#[repr(C, align(8))]
pub struct CloneArgs {
    pub flags:        u64,    /* CLONE_* (incl. upper bits) */
    pub pidfd:        u64,    /* &int — pidfd output */
    pub child_tid:    u64,    /* &pid_t */
    pub parent_tid:   u64,    /* &pid_t */
    pub exit_signal:  u64,
    pub stack:        u64,
    pub stack_size:   u64,
    pub tls:          u64,
    pub set_tid:      u64,    /* &pid_t[] */
    pub set_tid_size: u64,
    pub cgroup:       u64,    /* fd — for CLONE_INTO_CGROUP */
}

pub const CLONE_ARGS_SIZE_VER0: usize = 64;
pub const CLONE_ARGS_SIZE_VER1: usize = 80;
pub const CLONE_ARGS_SIZE_VER2: usize = 88;

const _ASSERT_CLONE_ARGS_VER2: () = assert!(core::mem::size_of::<CloneArgs>() == 88);
const _ASSERT_CLONE_ARGS_ALIGN: () = assert!(core::mem::align_of::<CloneArgs>() == 8);

// Scheduling policies
pub const SCHED_NORMAL: u32   = 0;
pub const SCHED_FIFO: u32     = 1;
pub const SCHED_RR: u32       = 2;
pub const SCHED_BATCH: u32    = 3;
/* 4 reserved (SCHED_ISO never implemented) */
pub const SCHED_IDLE: u32     = 5;
pub const SCHED_DEADLINE: u32 = 6;
pub const SCHED_EXT: u32      = 7;

pub const SCHED_RESET_ON_FORK: u32 = 0x40000000;

// SCHED_FLAG_* for sched_attr.sched_flags
pub const SCHED_FLAG_RESET_ON_FORK:  u64 = 0x01;
pub const SCHED_FLAG_RECLAIM:        u64 = 0x02;
pub const SCHED_FLAG_DL_OVERRUN:     u64 = 0x04;
pub const SCHED_FLAG_KEEP_POLICY:    u64 = 0x08;
pub const SCHED_FLAG_KEEP_PARAMS:    u64 = 0x10;
pub const SCHED_FLAG_UTIL_CLAMP_MIN: u64 = 0x20;
pub const SCHED_FLAG_UTIL_CLAMP_MAX: u64 = 0x40;

pub const SCHED_FLAG_KEEP_ALL: u64    = SCHED_FLAG_KEEP_POLICY | SCHED_FLAG_KEEP_PARAMS;
pub const SCHED_FLAG_UTIL_CLAMP: u64  = SCHED_FLAG_UTIL_CLAMP_MIN | SCHED_FLAG_UTIL_CLAMP_MAX;
pub const SCHED_FLAG_ALL: u64         = SCHED_FLAG_RESET_ON_FORK
                                       | SCHED_FLAG_RECLAIM
                                       | SCHED_FLAG_DL_OVERRUN
                                       | SCHED_FLAG_KEEP_ALL
                                       | SCHED_FLAG_UTIL_CLAMP;

pub const SCHED_GETATTR_FLAG_DL_DYNAMIC: u64 = 0x01;

#[repr(C, align(8))]
pub struct SchedAttr {
    pub size:           u32,
    pub sched_policy:   u32,
    pub sched_flags:    u64,
    pub sched_nice:     i32,
    pub sched_priority: u32,
    pub sched_runtime:  u64,
    pub sched_deadline: u64,
    pub sched_period:   u64,
    pub sched_util_min: u32,
    pub sched_util_max: u32,
}

pub const SCHED_ATTR_SIZE_VER0: usize = 48;
pub const SCHED_ATTR_SIZE_VER1: usize = 56;

const _ASSERT_SCHED_ATTR_VER1: () = assert!(core::mem::size_of::<SchedAttr>() == 56);
```

`SysClone::sys_clone3(args_user, size) -> Result<i64>`:
1. /* Size sanity */
2. if size < CLONE_ARGS_SIZE_VER0 || size > PAGE_SIZE || size % 8 != 0: return Err(EINVAL).
3. let mut args = CloneArgs::default();
4. let known = core::mem::size_of::<CloneArgs>();
5. if size <= known {
       copy_from_user_partial(&mut args, args_user, size)?;
   } else {
       copy_from_user(&mut args, args_user)?;
       /* Trailing bytes must be 0 */
       check_zero_user_tail(args_user.add(known), size - known)?;
   }
6. /* Flag validation */
7. validate_clone_flags(args.flags)?;
8. /* exit_signal sanity */
9. if args.exit_signal != 0 && args.exit_signal >= 128: return Err(EINVAL).
10. /* CLONE_THREAD ⟹ CLONE_SIGHAND ⟹ CLONE_VM */
11. if args.flags & CLONE_THREAD != 0 && args.flags & CLONE_SIGHAND == 0: return Err(EINVAL).
12. if args.flags & CLONE_SIGHAND != 0 && args.flags & CLONE_VM == 0: return Err(EINVAL).
13. /* CLONE_NEWPID ⊥ CLONE_THREAD */
14. if args.flags & CLONE_NEWPID != 0 && args.flags & CLONE_THREAD != 0: return Err(EINVAL).
15. /* CLONE_NEWUSER ⊥ CLONE_FS */
16. if args.flags & CLONE_NEWUSER != 0 && args.flags & CLONE_FS != 0: return Err(EINVAL).
17. /* CLONE_PIDFD_AUTOKILL ⟹ CLONE_PIDFD */
18. if args.flags & CLONE_PIDFD_AUTOKILL != 0 && args.flags & CLONE_PIDFD == 0: return Err(EINVAL).
19. /* Capability check for namespaces */
20. for ns_flag in [CLONE_NEWNS, CLONE_NEWUTS, CLONE_NEWIPC, CLONE_NEWCGROUP, CLONE_NEWNET, CLONE_NEWPID, CLONE_NEWTIME] {
        if args.flags & ns_flag != 0 && !ns_capable(current.user_ns, CAP_SYS_ADMIN): return Err(EPERM);
    }
21. /* CLONE_NEWUSER: unprivileged path gated by sysctl */
22. if args.flags & CLONE_NEWUSER != 0 && !sysctl::unprivileged_userns_clone() && !capable(CAP_SYS_ADMIN):
        return Err(EPERM);
23. /* Construct task */
24. Kernel::copy_process(args)

`SysSched::sys_sched_setattr(pid, attr_user, flags) -> Result<()>`:
1. if flags != 0: return Err(EINVAL).
2. let mut size_header: u32 = 0;
3. copy_from_user(&mut size_header, attr_user.as_ptr::<u32>())?;
4. if size_header < SCHED_ATTR_SIZE_VER0 as u32: return Err(E2BIG).
5. /* Trailing-zero check beyond known size */
6. let known = core::mem::size_of::<SchedAttr>() as u32;
7. let mut attr = SchedAttr::default();
8. if size_header <= known {
       copy_from_user_partial(&mut attr, attr_user, size_header as usize)?;
   } else {
       copy_from_user(&mut attr, attr_user)?;
       check_zero_user_tail(attr_user.add(known as usize), (size_header - known) as usize)?;
   }
9. attr.size = size_header;
10. /* Policy validation */
11. let policy = attr.sched_policy & !SCHED_RESET_ON_FORK;
12. if !matches!(policy, SCHED_NORMAL | SCHED_FIFO | SCHED_RR | SCHED_BATCH | SCHED_IDLE | SCHED_DEADLINE | SCHED_EXT):
        return Err(EINVAL).
13. /* Flag validation */
14. if attr.sched_flags & !SCHED_FLAG_ALL != 0: return Err(EINVAL).
15. /* Mutually-exclusive flags */
16. if attr.sched_flags & SCHED_FLAG_KEEP_POLICY != 0 && policy_change_intended(&attr):
        return Err(EINVAL).
17. /* Policy-specific param checks */
18. match policy {
        SCHED_FIFO | SCHED_RR => {
            if !(1..=99).contains(&attr.sched_priority): return Err(EINVAL);
            if !capable(CAP_SYS_NICE) && !within_rt_rlimit(attr.sched_priority): return Err(EPERM);
        }
        SCHED_DEADLINE => {
            if !capable(CAP_SYS_NICE): return Err(EPERM);
            if !(attr.sched_runtime <= attr.sched_deadline && attr.sched_deadline <= attr.sched_period):
                return Err(EINVAL);
            /* Admission control */
            dl_admission_check(&attr)?;
        }
        SCHED_NORMAL | SCHED_BATCH | SCHED_IDLE => {
            if !(-20..=19).contains(&attr.sched_nice): return Err(EINVAL);
            if attr.sched_priority != 0: return Err(EINVAL);
        }
        SCHED_EXT => {
            if !scx_enabled(): return Err(ENODEV);
        }
        _ => unreachable!(),
    }
19. /* Util clamp validation */
20. if attr.sched_flags & SCHED_FLAG_UTIL_CLAMP_MIN != 0 {
        if attr.sched_util_min > SCHED_CAPACITY_SCALE: return Err(EINVAL);
    }
21. if attr.sched_flags & SCHED_FLAG_UTIL_CLAMP_MAX != 0 {
        if attr.sched_util_max > SCHED_CAPACITY_SCALE: return Err(EINVAL);
        if attr.sched_util_min > attr.sched_util_max: return Err(EINVAL);
    }
22. /* DL_OVERRUN / RECLAIM gated on DEADLINE */
23. if attr.sched_flags & (SCHED_FLAG_DL_OVERRUN | SCHED_FLAG_RECLAIM) != 0 && policy != SCHED_DEADLINE:
        return Err(EINVAL).
24. /* Apply */
25. Scheduler::set_attr(pid, &attr)

### Out of Scope

- `kernel/fork.c::copy_process` — covered in `kernel/fork.md` Tier-3
- `kernel/sched/core.c` scheduler hot path — covered in `kernel/sched/core.md` Tier-3
- `kernel/sched/fair.c` CFS / EEVDF — covered in `kernel/sched/fair.md` Tier-3
- `kernel/sched/rt.c` SCHED_FIFO / SCHED_RR — covered in `kernel/sched/rt.md` Tier-3
- `kernel/sched/deadline.c` SCHED_DEADLINE / GRUB — covered in `kernel/sched/deadline.md` Tier-3
- `kernel/sched/ext.c` SCHED_EXT BPF scheduler class — covered in `kernel/sched/ext.md` Tier-3
- `kernel/nsproxy.c` namespace plumbing — covered in `kernel/nsproxy.md` Tier-3
- `kernel/user_namespace.c` userns uid/gid maps — covered in `kernel/user_namespace.md` Tier-3
- `kernel/cgroup/` cgroup v2 — covered in `kernel/cgroup/00-overview.md` Tier-3
- `sched_setaffinity(2)` / `sched_getaffinity(2)` CPU-mask details — covered in `uapi/syscalls/sched-affinity.md` Tier-5
- glibc `pthread_create`, NPTL — out of UAPI scope
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `CSIGNAL` | per-exit-signal low byte | shared |
| `CLONE_VM` / `_FS` / `_FILES` / `_SIGHAND` | per-share | shared |
| `CLONE_PIDFD` | per-pidfd allocation in parent | shared |
| `CLONE_PTRACE` / `_UNTRACED` | per-ptrace inheritance | shared |
| `CLONE_VFORK` | per-parent-suspend | shared |
| `CLONE_PARENT` / `_THREAD` | per-parent topology | shared |
| `CLONE_NEW{NS,UTS,IPC,USER,PID,NET,CGROUP,TIME}` | per-namespace creation | shared |
| `CLONE_SYSVSEM` | per-SysV semaphore undo share | shared |
| `CLONE_SETTLS` | per-TLS register set | shared |
| `CLONE_{PARENT,CHILD}_SETTID` / `CLONE_CHILD_CLEARTID` | per-TID copy/clear | shared |
| `CLONE_DETACHED` | per-historical (no-op) | shared |
| `CLONE_IO` | per-IO context share | shared |
| `CLONE_CLEAR_SIGHAND` | per-clone3 sighand reset | shared |
| `CLONE_INTO_CGROUP` | per-clone3 cgroup placement | shared |
| `CLONE_AUTOREAP` | per-clone3 auto-reap | shared |
| `CLONE_NNP` | per-clone3 no_new_privs | shared |
| `CLONE_PIDFD_AUTOKILL` | per-clone3 kill-on-pidfd-close | shared |
| `CLONE_EMPTY_MNTNS` | per-clone3 empty-mnt-ns | shared |
| `UNSHARE_EMPTY_MNTNS` | per-unshare | shared |
| `struct clone_args` | per-clone3 arg block | `CloneArgs` |
| `CLONE_ARGS_SIZE_VER0/1/2` | per-clone3 version | shared |
| `SCHED_NORMAL` / `_FIFO` / `_RR` / `_BATCH` / `_IDLE` / `_DEADLINE` / `_EXT` | per-policy | shared |
| `SCHED_RESET_ON_FORK` | per-policy modifier | shared |
| `SCHED_FLAG_*` | per-sched_attr flag | shared |
| `SCHED_GETATTR_FLAG_DL_DYNAMIC` | per-getattr output | shared |
| `struct sched_attr` | per-sched_{set,get}attr arg | `SchedAttr` |
| `SCHED_ATTR_SIZE_VER0/1` | per-sched_attr version | shared |

### abi surface (constants + structs)

### `CLONE_*` low-32 flags (`uapi/linux/sched.h:10-34`)

```text
CSIGNAL              = 0x000000ff   /* low 8 bits: exit signal */
CLONE_VM             = 0x00000100   /* share VM */
CLONE_FS             = 0x00000200   /* share fs info (cwd, umask) */
CLONE_FILES          = 0x00000400   /* share fd table */
CLONE_SIGHAND        = 0x00000800   /* share signal handlers + blocked */
CLONE_PIDFD          = 0x00001000   /* return pidfd in parent */
CLONE_PTRACE         = 0x00002000   /* parent's ptrace inherited */
CLONE_VFORK          = 0x00004000   /* parent suspends until child exec/exit */
CLONE_PARENT         = 0x00008000   /* same parent as cloner */
CLONE_THREAD         = 0x00010000   /* same thread group */
CLONE_NEWNS          = 0x00020000   /* new mount namespace */
CLONE_SYSVSEM        = 0x00040000   /* share SysV SEM_UNDO */
CLONE_SETTLS         = 0x00080000   /* set new TLS for child */
CLONE_PARENT_SETTID  = 0x00100000   /* store child TID in parent's memory */
CLONE_CHILD_CLEARTID = 0x00200000   /* clear child TID at thread exit (futex wake) */
CLONE_DETACHED       = 0x00400000   /* unused; ignored */
CLONE_UNTRACED       = 0x00800000   /* tracer cannot force CLONE_PTRACE */
CLONE_CHILD_SETTID   = 0x01000000   /* store child TID in child's memory */
CLONE_NEWCGROUP      = 0x02000000   /* new cgroup namespace */
CLONE_NEWUTS         = 0x04000000   /* new UTS namespace */
CLONE_NEWIPC         = 0x08000000   /* new IPC namespace */
CLONE_NEWUSER        = 0x10000000   /* new user namespace */
CLONE_NEWPID         = 0x20000000   /* new PID namespace */
CLONE_NEWNET         = 0x40000000   /* new network namespace */
CLONE_IO             = 0x80000000   /* share IO context */
```

### `CLONE_NEWTIME` and `UNSHARE_EMPTY_MNTNS` (`uapi/linux/sched.h:48, 54`)

```text
/* CLONE_NEWTIME sits inside CSIGNAL bits — usable only via unshare(2) and clone3(2). */
CLONE_NEWTIME        = 0x00000080   /* new time namespace */

/* unshare-only; overlaps CLONE_PARENT_SETTID bit position. */
UNSHARE_EMPTY_MNTNS  = 0x00100000   /* unshare into empty mount namespace */
```

### `clone3(2)`-only flags in the upper 32 bits (`uapi/linux/sched.h:37-42`)

```text
CLONE_CLEAR_SIGHAND   = (1ULL << 32)   /* clear all sig handlers, reset to SIG_DFL */
CLONE_INTO_CGROUP     = (1ULL << 33)   /* place child in cgroup specified by clone_args.cgroup */
CLONE_AUTOREAP        = (1ULL << 34)   /* auto-reap child on exit */
CLONE_NNP             = (1ULL << 35)   /* set no_new_privs on child */
CLONE_PIDFD_AUTOKILL  = (1ULL << 36)   /* kill child when its pidfd's last reference drops */
CLONE_EMPTY_MNTNS     = (1ULL << 37)   /* create an empty mount namespace */
```

### `struct clone_args` (`uapi/linux/sched.h:102-114`)

```text
struct clone_args {
    __aligned_u64 flags;          /* CLONE_* (incl. upper-32 bits) */
    __aligned_u64 pidfd;          /* &int — pidfd output if CLONE_PIDFD */
    __aligned_u64 child_tid;      /* &pid_t — for CLONE_CHILD_SETTID */
    __aligned_u64 parent_tid;     /* &pid_t — for CLONE_PARENT_SETTID */
    __aligned_u64 exit_signal;    /* signal to send to parent on exit */
    __aligned_u64 stack;          /* lowest address of child stack */
    __aligned_u64 stack_size;     /* size of child stack */
    __aligned_u64 tls;            /* TLS descriptor (for CLONE_SETTLS) */
    __aligned_u64 set_tid;        /* &pid_t[] — preselected PIDs */
    __aligned_u64 set_tid_size;   /* count of set_tid array */
    __aligned_u64 cgroup;         /* fd of cgroup (CLONE_INTO_CGROUP) */
};

CLONE_ARGS_SIZE_VER0 = 64    /* first published: through `tls` */
CLONE_ARGS_SIZE_VER1 = 80    /* + set_tid, set_tid_size */
CLONE_ARGS_SIZE_VER2 = 88    /* + cgroup */
```

Every field is `__aligned_u64` (8-byte aligned u64) — userspace `int *` and `pid_t *` pointers are zero-extended into u64 by the libc wrapper. New fields are added strictly at the end; the `size` argument to `clone3` tells the kernel how many trailing fields are valid.

### Scheduling policies (`uapi/linux/sched.h:124-134`)

```text
SCHED_NORMAL   = 0    /* CFS / EEVDF */
SCHED_FIFO     = 1    /* first-in-first-out RT */
SCHED_RR       = 2    /* round-robin RT */
SCHED_BATCH    = 3    /* CPU-bound batch (CFS variant) */
/* 4 reserved for SCHED_ISO (never implemented) */
SCHED_IDLE     = 5    /* very low priority (CFS variant) */
SCHED_DEADLINE = 6    /* EDF / CBS deadline scheduler */
SCHED_EXT      = 7    /* BPF scheduler class */

SCHED_RESET_ON_FORK = 0x40000000   /* OR into policy — reset to SCHED_NORMAL on fork */
```

`SCHED_RESET_ON_FORK` is a sticky bit ORed into the policy at `sched_setscheduler` time; it survives until cleared, and on every `fork(2)` the child's policy resets to `SCHED_NORMAL` and nice 0.

### `SCHED_FLAG_*` for `sched_attr.sched_flags` (`uapi/linux/sched.h:139-157`)

```text
SCHED_FLAG_RESET_ON_FORK    = 0x01    /* same semantics as SCHED_RESET_ON_FORK */
SCHED_FLAG_RECLAIM          = 0x02    /* SCHED_DEADLINE: allow reclaiming spare bandwidth */
SCHED_FLAG_DL_OVERRUN       = 0x04    /* SCHED_DEADLINE: send SIGXCPU on overrun */
SCHED_FLAG_KEEP_POLICY      = 0x08    /* don't change policy on sched_setattr */
SCHED_FLAG_KEEP_PARAMS      = 0x10    /* don't change runtime/deadline/period on sched_setattr */
SCHED_FLAG_UTIL_CLAMP_MIN   = 0x20    /* honor sched_util_min */
SCHED_FLAG_UTIL_CLAMP_MAX   = 0x40    /* honor sched_util_max */

SCHED_FLAG_KEEP_ALL    = SCHED_FLAG_KEEP_POLICY | SCHED_FLAG_KEEP_PARAMS  /* 0x18 */
SCHED_FLAG_UTIL_CLAMP  = SCHED_FLAG_UTIL_CLAMP_MIN | SCHED_FLAG_UTIL_CLAMP_MAX /* 0x60 */
SCHED_FLAG_ALL         = RESET_ON_FORK | RECLAIM | DL_OVERRUN | KEEP_ALL | UTIL_CLAMP   /* 0x7f */
```

### `sched_getattr` output-only flag (`uapi/linux/sched.h:160`)

```text
SCHED_GETATTR_FLAG_DL_DYNAMIC = 0x01   /* SCHED_DEADLINE bandwidth was dynamically adjusted */
```

### `struct sched_attr` (`uapi/linux/sched/types.h:98-119`)

```text
struct sched_attr {
    __u32 size;             /* sizeof(struct sched_attr) for fwd/bwd compat */

    __u32 sched_policy;     /* SCHED_* */
    __u64 sched_flags;      /* SCHED_FLAG_* */

    /* SCHED_NORMAL / SCHED_BATCH */
    __s32 sched_nice;       /* -20..19 */

    /* SCHED_FIFO / SCHED_RR */
    __u32 sched_priority;   /* 1..99 (real-time priority) */

    /* SCHED_DEADLINE (units = nanoseconds) */
    __u64 sched_runtime;
    __u64 sched_deadline;
    __u64 sched_period;

    /* Utilization clamping (since VER1) */
    __u32 sched_util_min;
    __u32 sched_util_max;
};

SCHED_ATTR_SIZE_VER0 = 48    /* first published: through sched_period */
SCHED_ATTR_SIZE_VER1 = 56    /* + sched_util_{min,max} */
```

`sched_util_min` / `sched_util_max` are values in `[0..SCHED_CAPACITY_SCALE = 1024]`, hinting CPU-frequency / placement; `-1` resets a clamp.

### compatibility contract

REQ-1: `clone(2)` first argument is `clone_flags` of width `unsigned long` (32-bit on i386, 64-bit on x86_64). On 32-bit archs, only the low 32 bits are reachable via `clone(2)`; `clone3(2)` is required to set bits 32+. The low 8 bits are `CSIGNAL` — the signal sent to the parent on child exit (usually `SIGCHLD`).

REQ-2: `CLONE_THREAD` requires `CLONE_SIGHAND`; `CLONE_SIGHAND` requires `CLONE_VM` (transitively, `CLONE_THREAD ⟹ CLONE_VM`). Violating these returns `-EINVAL`.

REQ-3: `CLONE_PARENT` and `CLONE_THREAD` are mutually exclusive with re-parenting init; both setting them returns `-EINVAL` if it would create an unreachable thread-group.

REQ-4: `CLONE_PIDFD` and `CLONE_THREAD` are mutually exclusive in `clone(2)`. `clone3(2)` allows both as of kernel 5.5+.

REQ-5: `CLONE_NEWUSER` placed before any other `CLONE_NEW*` in flag-processing order — the new user namespace becomes the context for capability checks on subsequent namespace creations. Combining `CLONE_NEWUSER` with `CLONE_FS` returns `-EINVAL` (changing fs root requires unshared FS).

REQ-6: `CLONE_NEWPID` cannot be combined with `CLONE_THREAD` (a thread cannot enter a new PID namespace). `-EINVAL`.

REQ-7: `CLONE_NEWNET`, `CLONE_NEWUTS`, `CLONE_NEWIPC`, `CLONE_NEWCGROUP`, `CLONE_NEWTIME`: each requires `CAP_SYS_ADMIN` in the user namespace owning the *target* namespace being unshared.

REQ-8: `CLONE_NEWUSER`: unprivileged user MAY create a new user namespace iff `sysctl kernel.unprivileged_userns_clone == 1` (default 0 in Rookery; 1 in distro stock Linux). When permitted, the new userns has a single uid mapping (caller's uid → uid 0) and full capabilities *within* that namespace.

REQ-9: `clone3(2)` argument struct size MUST be one of the published `CLONE_ARGS_SIZE_VER*` values OR a multiple-of-8 value greater than the highest known version (kernel zero-extends; any non-zero unknown trailing bytes return `-E2BIG`).

REQ-10: `struct clone_args` MUST be 8-byte aligned in user memory. Misaligned MUST return `-EINVAL`.

REQ-11: `clone_args.exit_signal` MUST be < 128 (signal-number range) or 0; out-of-range MUST return `-EINVAL`.

REQ-12: `CLONE_INTO_CGROUP` requires `clone_args.cgroup` to be a valid fd referencing a writable cgroup directory; caller MUST have write access to the target cgroup's `cgroup.threads` or `cgroup.procs`.

REQ-13: `CLONE_PIDFD` writes the new pidfd into `*clone_args.pidfd`. Cloexec is set automatically; bit is uninheritable across exec.

REQ-14: `CLONE_PIDFD_AUTOKILL` requires `CLONE_PIDFD` to be set; pidfd-close on the last reference sends `SIGKILL` to the child. The pidfd is treated as the *capability* to keep the child alive; useful for sidecar-process supervision.

REQ-15: `CLONE_NNP` sets `no_new_privs` on the child task, equivalent to `prctl(PR_SET_NO_NEW_PRIVS, 1)`. Cannot be undone after exec.

REQ-16: `CLONE_AUTOREAP` is the kernel-side equivalent of `SIGCHLD` + `SA_NOCLDWAIT` on the parent for this child only; the child's `task_struct` is reaped automatically.

REQ-17: Scheduling policies: `SCHED_NORMAL=0`, `SCHED_FIFO=1`, `SCHED_RR=2`, `SCHED_BATCH=3`, `SCHED_IDLE=5`, `SCHED_DEADLINE=6`, `SCHED_EXT=7`. Value 4 is reserved for `SCHED_ISO` and MUST return `-EINVAL`.

REQ-18: `SCHED_FIFO` and `SCHED_RR` require `CAP_SYS_NICE` (or rt-prio rlimit `RLIMIT_RTPRIO`). `sched_priority` MUST be in `[1, 99]`.

REQ-19: `SCHED_NORMAL` and `SCHED_BATCH` ignore `sched_priority`; `sched_nice` is the only knob. Range `[-20, 19]`.

REQ-20: `SCHED_DEADLINE` requires `CAP_SYS_NICE`; parameters `sched_runtime <= sched_deadline <= sched_period`; admission control enforces total per-CPU bandwidth.

REQ-21: `SCHED_EXT` policy requires a loaded BPF struct_ops scheduler; without one, set returns `-ENODEV`.

REQ-22: `SCHED_RESET_ON_FORK` MUST be honored: child of an RT task is reset to `SCHED_NORMAL` nice 0 on the next `fork(2)`/`clone(2)`. Same semantic via `SCHED_FLAG_RESET_ON_FORK`.

REQ-23: `struct sched_attr.size` MUST be `>= SCHED_ATTR_SIZE_VER0 (48)`. Smaller MUST return `-E2BIG`. Larger is permitted if all trailing bytes are 0 (forward-compat).

REQ-24: `struct sched_attr` MUST be 8-byte aligned (`__aligned_u64` semantics on `sched_flags`/`sched_runtime`/`sched_deadline`/`sched_period` requires this). Misaligned MUST return `-EFAULT`.

REQ-25: `SCHED_FLAG_UTIL_CLAMP_MIN` / `_MAX` require `sched_util_min <= sched_util_max <= SCHED_CAPACITY_SCALE` (1024). Combined with `SCHED_FLAG_KEEP_PARAMS`, util-clamp bounds may be updated without touching deadline params.

REQ-26: `SCHED_FLAG_DL_OVERRUN` (DEADLINE-only) MUST send `SIGXCPU` on bandwidth overrun. Combined with non-DEADLINE policy returns `-EINVAL`.

REQ-27: `SCHED_FLAG_RECLAIM` (DEADLINE-only) allows reclaiming spare bandwidth via GRUB algorithm. Combined with non-DEADLINE policy returns `-EINVAL`.

REQ-28: `SCHED_FLAG_KEEP_POLICY | SCHED_FLAG_KEEP_PARAMS` (= `SCHED_FLAG_KEEP_ALL`) means "modify only sched_flags and util-clamp"; pre-existing policy + params preserved.

REQ-29: `sched_getattr(2)` writes the user-supplied `size` field with the kernel's known size if user-supplied size > kernel size (telling user to allocate larger). If user-supplied size is too small, returns `-E2BIG`.

REQ-30: `sched_getattr(2)` MAY set `SCHED_GETATTR_FLAG_DL_DYNAMIC` in `sched_flags` to indicate the DEADLINE bandwidth was admission-control-clamped.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `clone3_size_aligned` | INVARIANT | per-clone3: `size >= 64 && size % 8 == 0` else `-EINVAL`. |
| `clone3_trailing_zero` | INVARIANT | per-clone3: trailing bytes beyond known size must be 0 else `-E2BIG`. |
| `clone_thread_implies_sighand_vm` | INVARIANT | per-clone: CLONE_THREAD ⟹ CLONE_SIGHAND ⟹ CLONE_VM. |
| `clone_newpid_excludes_thread` | INVARIANT | per-clone: CLONE_NEWPID ∧ CLONE_THREAD ⟹ `-EINVAL`. |
| `clone_newuser_excludes_fs` | INVARIANT | per-clone: CLONE_NEWUSER ∧ CLONE_FS ⟹ `-EINVAL`. |
| `clone_pidfd_autokill_requires_pidfd` | INVARIANT | per-clone3: CLONE_PIDFD_AUTOKILL ⟹ CLONE_PIDFD. |
| `clone_exit_signal_range` | INVARIANT | per-clone3: `exit_signal == 0 ∨ exit_signal ∈ [1,128)`. |
| `sched_attr_size_min` | INVARIANT | per-sched_setattr: `size >= SCHED_ATTR_SIZE_VER0` else `-E2BIG`. |
| `sched_attr_u64_alignment` | INVARIANT | `align_of::<SchedAttr>() == 8`. |
| `sched_policy_valid` | INVARIANT | per-sched_setattr: `policy ∈ {0,1,2,3,5,6,7}`; 4 (SCHED_ISO) ⟹ `-EINVAL`. |
| `sched_flag_subset_of_all` | INVARIANT | per-sched_setattr: `sched_flags & !SCHED_FLAG_ALL == 0`. |
| `sched_fifo_rr_prio_range` | INVARIANT | per-FIFO/RR: `sched_priority ∈ [1, 99]`. |
| `sched_normal_batch_nice_range` | INVARIANT | per-NORMAL/BATCH: `sched_nice ∈ [-20, 19]`. |
| `sched_deadline_invariant` | INVARIANT | per-DEADLINE: `runtime <= deadline <= period`. |
| `sched_util_clamp_order` | INVARIANT | per-CLAMP: `0 <= util_min <= util_max <= 1024`. |
| `sched_fifo_rr_requires_cap_sys_nice` | INVARIANT | per-FIFO/RR set: caller has CAP_SYS_NICE or within RLIMIT_RTPRIO. |
| `sched_deadline_requires_cap_sys_nice` | INVARIANT | per-DEADLINE set: caller has CAP_SYS_NICE. |
| `sched_ext_requires_scx` | INVARIANT | per-EXT: BPF struct_ops scheduler loaded or `-ENODEV`. |
| `clone_args_size_88` | INVARIANT | `sizeof::<CloneArgs>() == 88`. |

### Layer 2: TLA+

`uapi/headers/sched.tla`:
- Per-`clone3(2)` flag-consistency → namespace-creation → task duplication → pidfd issuance.
- Per-`sched_setattr(2)` policy-transition: NORMAL ↔ FIFO ↔ RR ↔ DEADLINE ↔ EXT.
- Per-DEADLINE admission control + GRUB reclaim.
- Per-UTIL_CLAMP propagation to EAS / DVFS hints.
- Properties:
  - `safety_no_thread_without_sighand` — per-clone: CLONE_THREAD without CLONE_SIGHAND never produces a thread.
  - `safety_no_pid_ns_for_thread` — per-clone: CLONE_NEWPID never creates a new PID ns for a thread.
  - `safety_user_ns_gates_other_ns_caps` — per-clone: CLONE_NEW{NS,UTS,IPC,NET,PID,CGROUP,TIME} require CAP_SYS_ADMIN in caller's user ns OR in the user ns created by CLONE_NEWUSER (processed first).
  - `safety_pidfd_autokill_releases_only_on_pidfd_drop` — per-clone3 with PIDFD_AUTOKILL: child SIGKILL fires iff *last* pidfd ref is dropped (not parent exit).
  - `safety_nnp_irrevocable` — per-CLONE_NNP child: subsequent exec cannot regain dropped privs.
  - `safety_sched_setattr_atomic_swap` — per-sched_setattr: policy+params transition is atomic (no observable intermediate).
  - `safety_dl_admission_total_bw_le_1` — per-DEADLINE: sum of `runtime/period` across all DEADLINE tasks ≤ admission cap.
  - `safety_reset_on_fork` — per-fork from RT task with RESET_ON_FORK: child is SCHED_NORMAL nice 0.
  - `liveness_clone3_returns_or_errors` — per-clone3: terminates with PID, `-EINVAL`, `-EPERM`, or `-ENOMEM`.
  - `liveness_sched_setattr_terminates` — per-sched_setattr: terminates with success or specific errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `CloneArgs` layout: 88 bytes, 8-byte aligned, fields in upstream order | `CloneArgs` |
| `SchedAttr` layout: 56 bytes, 8-byte aligned, u64 fields naturally aligned | `SchedAttr` |
| `sys_clone3` post: all REQ-2..16 enforced before `copy_process` | `sys_clone3` |
| `sys_sched_setattr` post: REQ-23..30 enforced before `Scheduler::set_attr` | `sys_sched_setattr` |
| `sys_sched_setattr` post: DEADLINE admission control invoked before commit | `sys_sched_setattr` |
| `sys_sched_setattr` post: util_clamp bounds checked before commit | `sys_sched_setattr` |
| `sys_sched_getattr` post: `size` field reflects kernel-known size on truncation | `sys_sched_getattr` |
| `sys_clone3` post: CLONE_INTO_CGROUP fd opened with write to `cgroup.threads`/`procs` | `sys_clone3` |
| `sys_clone3` post: CLONE_NEWUSER processed before sibling namespaces | `sys_clone3` |

### Layer 4: Verus/Creusot functional

`Per clone3(2) → validate(flags, size) → copy_process → return pidfd / pid` semantic equivalence: per-`clone3(2)` man page, per-`kernel/fork.c::kernel_clone3` upstream. `Per sched_setattr(2) → validate(size, policy, flags, params) → __sched_setscheduler` semantic equivalence: per-`sched_setattr(2)` man page, per-`kernel/sched/core.c::sched_setattr_nocheck`. `Per sched_getattr(2) → fill SchedAttr from current task → handle size negotiation` semantic equivalence: per-`Documentation/scheduler/sched-deadline.rst`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

Scheduler / clone UAPI reinforcement:

- **Flag consistency checked before any task allocation** — `CLONE_THREAD ⟹ CLONE_SIGHAND ⟹ CLONE_VM` enforced at syscall entry. Defense against per-half-shared task that would corrupt sighand/mm refcounts.
- **`clone3` trailing-zero check** — bytes beyond the highest known struct version MUST be 0, else `-E2BIG`. Defense against per-future-field silent activation.
- **`CLONE_NEWPID ⊥ CLONE_THREAD`** — defense against per-thread-in-new-PID-ns paradox.
- **`CLONE_NEWUSER ⊥ CLONE_FS`** — defense against per-userns-on-shared-fs root-pivot.
- **`CLONE_NEWUSER` gated by `kernel.unprivileged_userns_clone` sysctl** — defense against per-userns-mediated kernel-bug class (most of the historical local-priv-esc CVEs began with an unprivileged userns clone).
- **`CLONE_PIDFD_AUTOKILL` requires `CLONE_PIDFD`** — defense against per-orphaned-autokill semantic confusion.
- **`exit_signal < 128`** — defense against per-out-of-range signal that would underflow into kernel signal-number space.
- **`CLONE_INTO_CGROUP` cgroup fd write-checked** — defense against per-unauthorized-cgroup-placement.
- **`SCHED_FIFO` / `_RR` / `_DEADLINE` require `CAP_SYS_NICE`** — defense against per-unprivileged-RT-starvation DoS.
- **`SCHED_DEADLINE` admission control** — total bandwidth Σ(runtime/period) ≤ 1 per CPU; defense against per-overcommit deadline-miss cascade.
- **`SCHED_EXT` gated on loaded BPF scheduler** — defense against per-policy-without-implementation `-EINVAL`-handler crash.
- **`sched_attr.size` strict** — `>= SCHED_ATTR_SIZE_VER0`; defense against per-undersized-struct OOB-read.
- **`sched_flags & ~SCHED_FLAG_ALL`** rejected — defense against per-future-flag silent activation.
- **`SCHED_ISO` (value 4) reserved** — defense against per-policy-table OOB.

### grsecurity/pax-style reinforcement

- **GRKERNSEC_CHROOT_NICE** — a process inside a chroot OR inside an unprivileged user namespace MUST NOT call `setpriority(2)` to lower its nice value below 0, MUST NOT switch to `SCHED_FIFO` / `SCHED_RR` / `SCHED_DEADLINE`, MUST NOT set `SCHED_RESET_ON_FORK`, and MUST NOT call `sched_setaffinity(2)` to a CPU set wider than the cgroup's `cpuset.cpus`. Defeats per-chroot privilege-amplification via scheduler priority elevation.
- **PAX_RANDKSTACK at sched syscall entry** — randomizes kernel-stack offset per `sched_setscheduler`, `sched_setattr`, `sched_setaffinity`, `clone`, `clone3`, `unshare` entry. Defense against per-task_struct-pointer-leak through scheduler-side-channel disclosures (e.g., `sched_getattr` returning a pointer-shaped error path).
- **CAP_SYS_NICE for SCHED_FIFO / SCHED_RR / SCHED_DEADLINE in the *task's* user namespace** — strict: even if the caller holds CAP_SYS_NICE in its own user ns, switching another task to RT policies requires CAP_SYS_NICE in *that* task's user ns. Defeats per-cross-ns-priority-boost where a privileged outer-ns caller could push a low-priority container task into RT-starvation territory.
- **`sched_attr` u64 alignment requirement** — kernel rejects any `sched_attr_user` pointer that is not 8-byte aligned with `-EFAULT`, even on architectures that tolerate misalignment (x86_64, arm64). Defense against per-misaligned-copy partial-write race on weakly-ordered arches where `sched_runtime`, `sched_deadline`, `sched_period` could be observed half-written by another CPU.
- **`CLONE_NEWUSER` unprivileged-bridge** — Rookery sets `kernel.unprivileged_userns_clone = 0` by default. When the sysctl is flipped to 1, the *first* `CLONE_NEWUSER` from an unprivileged uid records the task under a per-uid quota (default 8 nested user namespaces per uid). Exceeding the quota returns `-EUSERS`. Defeats per-userns-bug-chain exploits (CVE-2022-32250-style) that depend on rapidly creating many namespaces.
- **`CLONE_NEWUSER` strict mapping rules** — the new user namespace's `uid_map` / `gid_map` MUST be installed before any privileged syscall in the new ns succeeds; the map MUST be a subset of the parent ns's owned uid range. PaX additionally refuses `setgroups` writes from the new userns until the parent has written `deny` (matches upstream behavior, with stricter enforcement on edge cases).
- **`CLONE_PIDFD_AUTOKILL` SIGKILL path goes through grsec audit** — every PIDFD-AUTOKILL signal is logged with the pid, uid, and pidfd-holder pid for forensic correlation. Defense against per-stealth-kill via pidfd lifecycle manipulation.
- **`CLONE_NNP` enforced for any setuid-inheriting exec** — Rookery enforces `CLONE_NNP` implicitly when the child is destined for a setuid binary that has dropped capabilities mid-exec, eliminating the per-exec privilege-regain race.
- **`set_tid` array bounded + per-PID-namespace validated** — `clone_args.set_tid` cannot request a PID already in use, cannot request PID < 2 (init reserved), and requires CAP_SYS_ADMIN in the *target* PID namespace. Defeats per-PID-prediction reconnaissance and per-init-impersonation attacks.
- **`clone3` flags upper-32-bit reserved-mask enforcement** — bits 38..63 MUST be 0. Defense against per-future-clone3-flag silent activation when running on a kernel that has not yet added support.
- **`SCHED_DEADLINE` admission control includes cgroup CPU-bandwidth attribution** — a task setting `SCHED_DEADLINE` consumes against both the global root_domain bandwidth AND its CPU cgroup's `cpu.max`. Defeats per-cgroup-bandwidth-bypass via DEADLINE policy.
- **`SCHED_EXT` BPF scheduler requires CAP_SYS_ADMIN + lockdown integrity mode** — switching the system to a BPF scheduler is treated as a kernel-state mutation requiring CAP_SYS_ADMIN; under `lockdown=integrity` or `lockdown=confidentiality` it is refused with `-EPERM`. Defeats per-BPF-scheduler kernel-rootkit class.

