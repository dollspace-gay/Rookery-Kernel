---
title: "Tier-5 syscall: getrusage(2) — syscall 98"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getrusage(2)` returns resource-usage statistics for the calling process, the entire thread-group's terminated children, or the calling thread alone. The `who` argument selects which scope; the kernel fills a `struct rusage` with per-CPU-time, per-memory, per-IO, per-context-switch, per-page-fault, and per-signal counters. Per-counters are decimated into a 16-field layout that has been ABI-stable since classical UNIX, with the Linux extension that several fields (`ru_idrss`, `ru_isrss`, `ru_msgsnd`, `ru_msgrcv`, `ru_nsignals`) are reserved-but-unmaintained.

Critical for: shells reporting `time` builtin output, `wait4(2)` consumers, profiling tools, container runtimes counting per-task CPU usage, RUSAGE_THREAD for thread-local accounting.

### Acceptance Criteria

- [ ] AC-1: `getrusage(RUSAGE_SELF, &r)` returns 0; `r.ru_utime + r.ru_stime` non-zero after non-trivial work.
- [ ] AC-2: `getrusage(RUSAGE_CHILDREN, &r)` initially returns zeros; after a `wait4`-reaped child, accumulated.
- [ ] AC-3: `getrusage(RUSAGE_THREAD, &r)` returns per-thread cputime.
- [ ] AC-4: `who = 2` (or anything else) → `-EINVAL`.
- [ ] AC-5: `r_usage = NULL` → `-EFAULT`.
- [ ] AC-6: After a major-fault loop, `ru_majflt` increments accordingly.
- [ ] AC-7: After a yield-loop, `ru_nvcsw` increments.
- [ ] AC-8: `ru_maxrss` non-decreasing across calls within RUSAGE_SELF.
- [ ] AC-9: Unmaintained fields (`ru_idrss`, `ru_isrss`, etc.) always zero.

### Architecture

```rust
#[syscall(nr = 98, abi = "sysv")]
pub fn sys_getrusage(who: i32, r_usage: UserPtrMut<Rusage>) -> isize {
    Sched::do_getrusage(who, r_usage)
}
```

`Sched::do_getrusage(who, r_usage) -> isize`:
1. let kind = match who {
   - 0  => RusageKind::Self_,
   - -1 => RusageKind::Children,
   - 1  => RusageKind::Thread,
   - _  => return -EINVAL,
   };
2. let mut r = Rusage::zeroed();
3. Sched::collect_rusage(&mut r, kind);
4. r_usage.copy_out(&r)?;                                // EFAULT
5. 0

`Sched::collect_rusage(r, kind) -> ()`:
1. let t = current();
2. let sig = t.signal;
3. match kind {
   - RusageKind::Self_ => {
     - /* Thread-group aggregation */
     - let (utime, stime) = thread_group_cputime_adjusted(t);
     - r.ru_utime = ktime_to_timeval(utime);
     - r.ru_stime = ktime_to_timeval(stime);
     - r.ru_maxrss = (t.mm.map(|m| m.hiwater_rss()).unwrap_or(0) * PAGE_SIZE_KB) as i64;
     - r.ru_minflt = sig.min_flt as i64;
     - r.ru_majflt = sig.maj_flt as i64;
     - r.ru_inblock = sig.io.read_bytes >> 9;
     - r.ru_oublock = sig.io.write_bytes >> 9;
     - r.ru_nvcsw = sig.nvcsw as i64;
     - r.ru_nivcsw = sig.nivcsw as i64;
     },
   - RusageKind::Children => {
     - r.ru_utime = ktime_to_timeval(sig.cutime);
     - r.ru_stime = ktime_to_timeval(sig.cstime);
     - r.ru_maxrss = sig.cmaxrss as i64;
     - r.ru_minflt = sig.cmin_flt as i64;
     - r.ru_majflt = sig.cmaj_flt as i64;
     - r.ru_inblock = sig.cinblock as i64;
     - r.ru_oublock = sig.coublock as i64;
     - r.ru_nvcsw = sig.cnvcsw as i64;
     - r.ru_nivcsw = sig.cnivcsw as i64;
     },
   - RusageKind::Thread => {
     - let (utime, stime) = task_cputime_adjusted(t);
     - r.ru_utime = ktime_to_timeval(utime);
     - r.ru_stime = ktime_to_timeval(stime);
     - r.ru_maxrss = (t.mm.map(|m| m.hiwater_rss()).unwrap_or(0) * PAGE_SIZE_KB) as i64;
     - r.ru_minflt = t.min_flt as i64;
     - r.ru_majflt = t.maj_flt as i64;
     - r.ru_inblock = t.ioac.read_bytes >> 9;
     - r.ru_oublock = t.ioac.write_bytes >> 9;
     - r.ru_nvcsw = t.nvcsw as i64;
     - r.ru_nivcsw = t.nivcsw as i64;
     },
   }
4. /* Reserved fields remain 0 */

### Out of Scope

- `wait4(2)` (covered in its own Tier-5 doc; shares accounting machinery).
- `prlimit64(2)` for setting/getting rlimits (covered separately).
- `times(2)` (covered in this wave's `times.md`).
- `/proc/<pid>/stat`, `/proc/<pid>/io` (covered under proc Tier-3).
- Per-cgroup CPU accounting (covered under cgroup Tier-3).
- Implementation code.

### signature

```c
int getrusage(int who, struct rusage *r_usage);
```

```c
#define RUSAGE_SELF     0
#define RUSAGE_CHILDREN (-1)
#define RUSAGE_THREAD   1   /* Linux-specific */

struct rusage {
    struct timeval ru_utime;    /* user CPU time used */
    struct timeval ru_stime;    /* system CPU time used */
    long           ru_maxrss;   /* maximum resident set size (KB) */
    long           ru_ixrss;    /* (unused) integral shared text memory size */
    long           ru_idrss;    /* (unused) integral unshared data size */
    long           ru_isrss;    /* (unused) integral unshared stack size */
    long           ru_minflt;   /* page reclaims (soft page faults) */
    long           ru_majflt;   /* page faults (hard page faults) */
    long           ru_nswap;    /* (unused) swaps */
    long           ru_inblock;  /* block input operations */
    long           ru_oublock;  /* block output operations */
    long           ru_msgsnd;   /* (unused) IPC messages sent */
    long           ru_msgrcv;   /* (unused) IPC messages received */
    long           ru_nsignals; /* (unused) signals received */
    long           ru_nvcsw;    /* voluntary context switches */
    long           ru_nivcsw;   /* involuntary context switches */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `who` | `int` | in | One of `RUSAGE_SELF`, `RUSAGE_CHILDREN`, `RUSAGE_THREAD`. |
| `r_usage` | `struct rusage *` | out | Caller-provided buffer; kernel writes 16 longs + 2 timevals. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; `r_usage` filled. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `r_usage` user pointer faults during copy_to_user. |
| `EINVAL` | `who` not one of the three legal values. |

### abi surface

```text
__NR_getrusage (x86_64) =  98
__NR_getrusage (arm64)  = 165
__NR_getrusage (riscv)  = 165
__NR_getrusage (i386)   =  77

/* struct rusage layout is fixed since 2.0; new fields would break ABI. */
/* timeval = { time_t tv_sec; suseconds_t tv_usec; } — on 32-bit the
   time_t is 32-bit, and y2038 is mitigated only via not actually
   wrapping (CPU time is delta-based, not absolute). */
```

### compatibility contract

REQ-1: Syscall number is **98** on x86_64. ABI-stable since 2.0.

REQ-2: `who` validation: `RUSAGE_SELF (0)`, `RUSAGE_CHILDREN (-1)`, `RUSAGE_THREAD (1)`; any other value → `-EINVAL`.

REQ-3: `RUSAGE_SELF`: aggregates `current->signal->utime`, `stime`, and all live thread members of the thread-group.

REQ-4: `RUSAGE_CHILDREN`: returns `current->signal->cutime`, `cstime`, and per-counters accumulated from `wait4(2)`-reaped children only (not still-live children).

REQ-5: `RUSAGE_THREAD`: returns the calling thread's own `utime`, `stime`, etc., not the thread-group.

REQ-6: `ru_utime`, `ru_stime`: computed via `task_cputime_adjusted()` (with CFS adjustment if needed) and converted from `cputime_t` (nanoseconds) to `struct timeval` (microseconds).

REQ-7: `ru_maxrss`: high-water-mark of resident set size in kilobytes. For `RUSAGE_CHILDREN`, the max across terminated children. For `RUSAGE_SELF`/`THREAD`, current high-water as recorded in `mm->hiwater_rss`.

REQ-8: `ru_minflt`, `ru_majflt`: page-fault counters from `task->signal->maj_flt` / `min_flt`. For `THREAD`, the per-thread counters.

REQ-9: `ru_inblock`, `ru_oublock`: aggregated from per-task `task_io_accounting` (`read_bytes` / `write_bytes` divided by 512).

REQ-10: `ru_nvcsw`, `ru_nivcsw`: voluntary / involuntary context-switch counters.

REQ-11: `ru_ixrss`, `ru_idrss`, `ru_isrss`, `ru_nswap`, `ru_msgsnd`, `ru_msgrcv`, `ru_nsignals`: filled with zero (reserved-but-unmaintained per POSIX).

REQ-12: Per-locking: the kernel uses `task_lock(current)` only when necessary; cputime queries use the `vtime`/`cputime_t` lockless reads where supported.

REQ-13: Per-32-bit-compat: `struct compat_rusage` with `compat_timeval`; `compat_getrusage` translates layout.

REQ-14: `wait4(2)` uses the same accounting machinery (`getrusage_*` helpers).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `who_total_dispatch` | INVARIANT | who not in {-1,0,1} ⟹ EINVAL. |
| `reserved_fields_zero` | INVARIANT | ru_idrss, ru_isrss, ru_nswap, ru_msgsnd, ru_msgrcv, ru_nsignals == 0. |
| `cputime_monotonic_per_call` | INVARIANT | per-task: subsequent getrusage call returns ru_utime/stime >= previous. |
| `children_only_reaped` | INVARIANT | RUSAGE_CHILDREN excludes still-live children. |
| `thread_scope_correct` | INVARIANT | RUSAGE_THREAD uses task cputime not signal cputime. |
| `maxrss_non_decreasing_self` | INVARIANT | ru_maxrss non-decreasing within RUSAGE_SELF. |

### Layer 2: TLA+

`kernel/getrusage.tla`:
- States: per-call scope-select, per-counter read, per-aggregation, per-copy_out.
- Properties:
  - `safety_scope_isolation` — RUSAGE_THREAD never touches signal-aggregate counters.
  - `safety_children_only_reaped` — RUSAGE_CHILDREN sees only `wait4`-reaped accumulations.
  - `safety_no_partial_copy` — copy_out is all-or-nothing.
  - `liveness_call_terminates` — getrusage always returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getrusage` post: returns 0 ∨ -EINVAL ∨ -EFAULT | `Sched::do_getrusage` |
| `collect_rusage` post: reserved fields zero | `Sched::collect_rusage` |
| `thread_group_cputime_adjusted` post: utime+stime >= prev call | `Sched::thread_group_cputime_adjusted` |

### Layer 4: Verus / Creusot functional

Per-`getrusage(2)` man-page + POSIX-2008 semantic equivalence (extended with RUSAGE_THREAD Linux extension). Selftests: `tools/testing/selftests/timers/` and `tools/testing/selftests/proc/` cputime subsets pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getrusage(2)` reinforcement:

- **Per-`who` strict dispatch** — defense against per-undefined-scope behavior.
- **Per-reserved-fields zeroed unconditionally** — defense against per-stale-stack info-leak.
- **Per-`task_cputime_adjusted` lockless read** — defense against per-cputime read-vs-update race.
- **Per-`hiwater_rss` monotonic** — defense against per-counter regression.
- **Per-`copy_to_user` all-or-nothing** — defense against per-partial-write info-leak.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `r_usage` copy_to_user** — defense against per-user-pointer kernel-deref bug; SMAP forced; stac/clac gated.
- **GRKERNSEC_PROC_GETPID** — `RUSAGE_CHILDREN` and `RUSAGE_SELF` accounting visible only to the owning UID (or root); defense against per-resource-usage fingerprinting info-leak when the same syscall can be invoked via `/proc/<pid>/stat` shadowing. Non-owners get zeroed counters for filtered tasks.
- **GRKERNSEC_HIDESYM on sched_clock-based cputime** — defense against per-sched_clock side-channel; cputime values rounded to a coarser granularity (1ms) when caller's UID does not own the target. The `ru_utime` / `ru_stime` becomes a quantized view to suppress fine-grained timing leaks of crypto / branch-prediction patterns.
- **PAX_USERCOPY_HARDEN on `struct rusage` copy_to_user** — bounded copy uses whitelisted slab region; defense against per-overlong-copy.
- **GRKERNSEC_NO_GETPID-style** — `RUSAGE_CHILDREN` over a chroot/userns boundary returns zeroed counters; defense against per-container info-leak about external workload.
- **PaX KERNEXEC on cputime conversion (`ktime_to_timeval`)** — defense against per-W^X violation in conversion-helper inlining path.
- **Per-`ru_maxrss` truncated under hardened policy** — when the caller's UID does not own the target, ru_maxrss is rounded to nearest 4MB; defense against per-RSS-fingerprinting via repeated polling.
- **PAX_REFCOUNT on `task_struct->signal` access via `task_lock`** — defense against per-task UAF race when target is concurrently exiting.
- **Per-`RUSAGE_THREAD` requires per-thread permission** — defense against per-cross-thread leak; only `ptrace`-permitted callers can read another thread's rusage via `proc` path (this syscall always self-scoped, but the proc shadow is filtered).
- **GRKERNSEC_HIDESYM also blanks `ru_nvcsw`/`ru_nivcsw` for non-owners** — defense against per-scheduler-fingerprinting info-leak.

