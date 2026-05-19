---
title: "Tier-5 syscall: sched_rr_get_interval(2) — syscall 148"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sched_rr_get_interval(2)` reports the **time-slice quantum** that the kernel would assign to a task running under SCHED_RR (round-robin). For SCHED_RR specifically the returned value is the configured `RR_TIMESLICE` (default `100 ms`, settable per-system via `/proc/sys/kernel/sched_rr_timeslice_ms`). For SCHED_FIFO the syscall returns `{0, 0}` (FIFO has no quantum). For SCHED_NORMAL / SCHED_BATCH the value reported is the per-task CFS estimate from `get_rr_interval_fair()` — historically a documentation-only approximation, sometimes zero. For SCHED_IDLE and SCHED_DEADLINE the kernel returns zero. The result is a `struct timespec` with seconds + nanoseconds resolution. Critical for: RT applications that need to size deadlines against the kernel's RR quantum; observability tooling (`chrt -p`); CFS bandwidth-tuning diagnostics; embedded systems that pin RR_TIMESLICE to a known value at boot.

This Tier-5 covers the userspace ABI of syscall 148.

### Acceptance Criteria

- [ ] AC-1: `sched_rr_get_interval(0, &tp)` for SCHED_RR self: returns `0`; `tp.tv_sec * 1e9 + tp.tv_nsec == sysctl_sched_rr_timeslice_ms * 1e6`.
- [ ] AC-2: `sched_rr_get_interval(0, &tp)` for SCHED_FIFO self: returns `0`; `tp == {0, 0}`.
- [ ] AC-3: `sched_rr_get_interval(0, &tp)` for SCHED_IDLE self: returns `0`; `tp == {0, 0}`.
- [ ] AC-4: `sched_rr_get_interval(0, &tp)` for SCHED_DEADLINE self: returns `0`; `tp == {0, 0}`.
- [ ] AC-5: `sched_rr_get_interval(0, &tp)` for SCHED_NORMAL self: returns `0`; `tp.tv_*` is the CFS estimate.
- [ ] AC-6: After `echo 50 > /proc/sys/kernel/sched_rr_timeslice_ms`: subsequent SCHED_RR call reports 50 ms.
- [ ] AC-7: `sched_rr_get_interval(-1, &tp)`: returns `-EINVAL`.
- [ ] AC-8: `sched_rr_get_interval(non_existent_pid, &tp)`: returns `-ESRCH`.
- [ ] AC-9: `sched_rr_get_interval(0, NULL)`: returns `-EFAULT`.
- [ ] AC-10: `tp.tv_nsec` is always normalised to `[0, 999_999_999]`.

### Architecture

Rookery surface in `kernel/sched/syscalls.rs`:

```rust
pub fn sys_sched_rr_get_interval(pid: i32, tp: UserPtrMut<Timespec>) -> isize {
    if pid < 0 {
        return -EINVAL as isize;
    }
    let target = match Task::find_in_pidns(current().pid_ns(), pid) {
        Some(t) => t,
        None    => return -ESRCH as isize,
    };
    let interval_ns = {
        let _g = target.rq_lock_irqsave();
        match target.policy() {
            SCHED_RR => sysctl::SCHED_RR_TIMESLICE_MS.load() as u64 * NSEC_PER_MSEC,
            SCHED_NORMAL | SCHED_BATCH => Sched::cfs_get_rr_interval(&target),
            _ => 0u64,
        }
    };
    let ts = Timespec {
        tv_sec:  (interval_ns / NSEC_PER_SEC) as i64,
        tv_nsec: (interval_ns % NSEC_PER_SEC) as i64,
    };
    match tp.copy_to_user(&ts) {
        Ok(_)  => 0,
        Err(_) => -EFAULT as isize,
    }
}
```

CFS estimate (per-`get_rr_interval_fair`):

```rust
fn cfs_get_rr_interval(task: &Task) -> u64 {
    /* ns timeslice ≈ task->se.load.weight / cfs_rq.load.weight * sched_latency_ns */
    let load_w  = task.se.load.weight as u64;
    let total_w = task.cfs_rq().load.weight.max(1) as u64;
    let latency = sysctl::SCHED_LATENCY_NS.load() as u64;
    latency * load_w / total_w
}
```

### Out of Scope

- `kernel/sched/rt.md` Tier-3: RR class internals, `RR_TIMESLICE` mechanics.
- `kernel/sched/fair.md` Tier-3: CFS `get_rr_interval` estimate computation.
- `sched_setscheduler.md` / `sched_setattr.md` siblings.
- `sched_rr_get_interval_time64.md` (32-bit compat variant).
- glibc / musl userspace wrappers.
- Implementation code.

### signature

```c
int sched_rr_get_interval(pid_t pid, struct timespec *tp);

struct timespec {
    time_t tv_sec;   /* seconds */
    long   tv_nsec;  /* nanoseconds [0, 999_999_999] */
};
```

Rust ABI shim:

```rust
pub fn sys_sched_rr_get_interval(pid: i32, tp: *mut Timespec) -> isize;
```

Syscall number: **148** (x86_64). Generic syscall table: **127**. Note there is a `sched_rr_get_interval_time64` variant on 32-bit architectures (syscall 423) that uses `struct __kernel_timespec`; this design covers the 64-bit native variant.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `pid` | `pid_t` | IN | Target task in caller's pid-ns; `0` = self. |
| `tp` | `struct timespec *` | OUT (UAPI ptr) | Pointer to user buffer; receives the RR interval via `copy_to_user`. |

### return

- **Success**: `0`; `*tp` populated.
- **Failure**: `-1` and `errno`; `*tp` not modified.

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `pid < 0`; `tp == NULL` (some kernels report this as `-EFAULT`). |
| `ESRCH` | No task with given pid in caller's pid-ns. |
| `EFAULT` | `tp` points to unmapped userspace memory. |

### abi surface

```text
__NR_sched_rr_get_interval (x86_64)             = 148
__NR_sched_rr_get_interval (i386)               = 161
__NR_sched_rr_get_interval (arm64)              = 127  (generic-syscall)
__NR_sched_rr_get_interval (generic)            = 127
__NR_sched_rr_get_interval_time64 (32-bit only) = 423

/* Return semantics by policy */
SCHED_RR              -> RR_TIMESLICE (default 100ms, /proc/sys/kernel/sched_rr_timeslice_ms)
SCHED_FIFO            -> {0, 0}
SCHED_NORMAL / BATCH  -> get_rr_interval_fair() estimate (often non-zero, see CFS)
SCHED_IDLE            -> {0, 0}
SCHED_DEADLINE        -> {0, 0}

struct timespec {
    time_t  tv_sec;
    long    tv_nsec;
};
```

### compatibility contract

REQ-1: Syscall number is **148** on x86_64, **127** on arm64/generic. ABI-stable since 2.0.

REQ-2: `pid == 0` ⟹ target current task.

REQ-3: `pid < 0` ⟹ `-EINVAL`.

REQ-4: Lookup uses caller's pid-ns; not-found ⟹ `-ESRCH`.

REQ-5: Dispatch to per-class `get_rr_interval(task)`:
- SCHED_RR: `RR_TIMESLICE_NS / NSEC_PER_TICK` ticks → ns; default 100 ms.
- SCHED_FIFO: zero.
- SCHED_NORMAL / SCHED_BATCH: `sched_class.get_rr_interval(rq, task)` from `kernel/sched/fair.c`; an estimate of `nice`-weighted timeslice.
- SCHED_IDLE: zero.
- SCHED_DEADLINE: zero.

REQ-6: `RR_TIMESLICE` is tunable via `/proc/sys/kernel/sched_rr_timeslice_ms` (range 1..INT_MAX ms). Default `100` ms.

REQ-7: `tp` populated via single `copy_to_user(tp, &interval, sizeof(struct timespec))`; atomic from user's perspective per `copy_to_user` semantics.

REQ-8: No capability check.

REQ-9: Read-only on task state; under `rq_lock` briefly for safe class dispatch.

REQ-10: Inheritance: irrelevant (read-only).

REQ-11: Atomicity wrt concurrent `sched_setscheduler`: the read returns the interval consistent with whichever policy was observed under `rq_lock`.

REQ-12: `tv_nsec` normalised to `[0, 999_999_999]`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tv_nsec_normalised` | INVARIANT | output `tv_nsec ∈ [0, 999_999_999]`. |
| `fifo_returns_zero` | INVARIANT | SCHED_FIFO ⟹ `tp == {0,0}`. |
| `rr_returns_rr_timeslice` | INVARIANT | SCHED_RR ⟹ `interval == sysctl_sched_rr_timeslice_ms * 1e6 ns`. |
| `copy_to_user_fault_safe` | INVARIANT | bad `tp` ⟹ `-EFAULT`; `tp` not partially written (`copy_to_user` is best-effort, but the syscall reports `EFAULT` only on full failure). |
| `read_only_task_state` | INVARIANT | target state unchanged. |

### Layer 2: TLA+

`uapi/sched-rr-get-interval.tla`:
- Variables: `target.policy`, `sysctl.rr_timeslice_ms`, `tp_out`.
- Properties:
  - `safety_class_interval_correct` — output matches per-policy table.
  - `safety_rr_tracks_sysctl` — SCHED_RR ⟹ output follows `sysctl_sched_rr_timeslice_ms`.
  - `safety_no_state_mutation` — task state unchanged.
  - `liveness_terminates` — call returns synchronously.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_sched_rr_get_interval` post: `tv_nsec ∈ [0, 1e9)` | `sys_sched_rr_get_interval` |
| `sys_sched_rr_get_interval` post for SCHED_RR: interval matches sysctl | `sys_sched_rr_get_interval` |
| `cfs_get_rr_interval` post: returns `latency_ns * load_w / total_w` | `cfs_get_rr_interval` |

### Layer 4: Verus/Creusot functional

Per-`sched_rr_get_interval(2)` man page; per-LTP `testcases/kernel/syscalls/sched_rr_get_interval/*`; per-glibc `sysdeps/unix/sysv/linux/sched_rr_get_interval.c`; per-`Documentation/scheduler/sched-rt-group.rst`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

sched_rr_get_interval reinforcement:

- **Per-rq_lock taken during class dispatch** — defense against per-class-switch race producing inconsistent interval.
- **Per-tv_nsec normalisation strict** — defense against per-malformed-timespec consumer (POSIX assumes normalised).
- **Per-policy table exhaustive** — defense against per-non-RR class falling through to RR_TIMESLICE.

### grsecurity/pax-style reinforcement

- **PaX UDEREF strict on `tp` write** — every `copy_to_user(tp)` goes through validated UDEREF helper; defense against attacker-set-up kernel pointer aliased as user.
- **PAX_RANDKSTACK on every entry** — randomise kstack on each `sched_rr_get_interval` entry; defense against per-stack-spray.
- **GRKERNSEC_HIDESYM on `sched_class` table pointer** — class-dispatch indirection excluded from `/proc/kallsyms` under `kptr_restrict ≥ 2`; defense against per-recon of scheduler-class internal layout.
- **GRKERNSEC_PROC_USER restrict cross-user readback** — under hardened policy, an unprivileged user can read interval only for own tasks; defense against per-RT-topology probing.
- **`/proc/sys/kernel/sched_rr_timeslice_ms` write restricted** — only `CAP_SYS_ADMIN` may mutate; this syscall reads the value, so its disclosure is bounded.
- **Audit per-cross-pid probe** — `sched_rr_get_interval(target_other_uid_pid, ...)` audit-logged as RT-topology reconnaissance.
- **Refuse `pid == 0` from sandboxed users** — under hardened policy, sandboxed users may read interval only for own pid (not 0-meaning-self alias); defense-in-depth observability gating.
- **Timing-side-channel mitigation** — return latency is independent of `target.policy` to defeat policy-probing via timing.
- **Deprecated-syscall ENOSYS gate (off by default)** — grsec hardened policy may force `sched_rr_get_interval` to `-ENOSYS`, forcing applications to read `/proc/sys/kernel/sched_rr_timeslice_ms` directly; defense against per-legacy-API surface.
- **Coarse-grain interval under RANDPID** — grsec policy may quantize the returned `tv_nsec` to the nearest tick boundary; defense against per-RT-quantum-probe used as a covert-channel.
- **GRKERNSEC_CLOCK_RESOLUTION** — pair with this syscall to clamp `tv_nsec` resolution exposed to userspace; defense against per-high-resolution timestamp side-channel.

