---
title: "Tier-5 syscall: nice(2) — syscall 34"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`nice(2)` is the **legacy / wrapper** API for adjusting the calling thread's nice value by a relative `inc`rement. Internally it is implemented as `setpriority(PRIO_PROCESS, 0, current_nice + inc)` with the result clamped to [-20, 19]. The historical wire ABI returned the **old** value (often 0) on success and -1 on error, leading to the same errno-vs-valid-value confusion as `getpriority(2)`; modern glibc wraps so userspace sees the **new** nice on success or -1 + errno on failure. Note that **arm64 / generic-syscall architectures do NOT expose `nice` as a syscall**; glibc emulates via setpriority. Critical for: `nice(1)` userspace tool, legacy daemons, batch-scheduler hooks, Rookery's POSIX-conformance shim.

### Acceptance Criteria

- [ ] AC-1: `nice(5)` from a nice-0 task returns 5; getpriority confirms.
- [ ] AC-2: `nice(0)` returns the current nice unchanged.
- [ ] AC-3: `nice(-5)` from a nice-0 task without CAP_SYS_NICE and RLIMIT_NICE=20 → -EPERM.
- [ ] AC-4: `nice(-5)` with RLIMIT_NICE=30 → -5.
- [ ] AC-5: `nice(100)` clamps and returns 19.
- [ ] AC-6: `nice(-100)` clamps then requires CAP_SYS_NICE; returns -20 or -EPERM.
- [ ] AC-7: Repeated `nice(1)` saturates at 19.
- [ ] AC-8: LSM denial → -EACCES.
- [ ] AC-9: `nice(5)` on a SCHED_FIFO task updates nominal nice; rt_priority unchanged.
- [ ] AC-10: After `nice(N)`, getpriority(PRIO_PROCESS, 0) returns current+N (clamped).
- [ ] AC-11: nice is interruptible by signal between schedule points (rare; mostly atomic).
- [ ] AC-12: On arm64, the libc-emulated path produces identical results.

### Architecture

```rust
#[syscall(nr = 34, abi = "sysv")]
pub fn sys_nice(inc: i32) -> SysResult<i32> {
    Nice::do_call(inc)
}
```

`Nice::do_call(inc) -> i32`:
1. let cur = task_nice(current());
2. /* Clamp new nice */
3. let mut new_nice = cur.saturating_add(inc);
4. new_nice = new_nice.clamp(NICE_MIN, NICE_MAX);
5. /* Sandbox gate (Rookery): clamp under sandbox flags */
6. new_nice = Nice::sandbox_clamp(current(), new_nice);
7. /* Permission gate */
8. if new_nice < cur {
       if !capable(CAP_SYS_NICE) {
           let cap = NICE_MAX - task_rlimit(current(), RLIMIT_NICE) as i32 + 1;
           if new_nice < cap { return -EPERM; }
       }
   }
9. /* LSM */
10. security_task_setnice(current(), new_nice)?;    // -EACCES
11. /* Apply */
12. Sched::set_user_nice(current(), new_nice);
13. /* Wire ABI: return new nice directly */
14. new_nice

`Nice::sandbox_clamp(task, new_nice) -> i32`:
1. if task_has_sandbox(task) {
       let floor = task.sandbox_nice_floor;          // e.g., 0 by default
       return new_nice.max(floor);
   }
2. new_nice

### Out of Scope

- `setpriority(2)` (Tier-5 separate doc — superset API).
- `getpriority(2)` (Tier-5 separate doc).
- CFS weight / load tracking (Tier-3 in `kernel/sched-fair.md`).
- `nice(1)` userspace tool (out of kernel scope).
- Implementation code.

### signature

```c
int nice(int inc);
```

### parameters

| Parameter | Type | Direction | Description |
|---|---|---|---|
| `inc` | `int` | in | Relative nice increment; positive = deprioritize, negative = priority elevation (privileged). |

### return value

| Value | Meaning |
|---|---|
| New nice value in [-20, 19] | Success (via glibc wrapper convention). |
| Pre-glibc-2.2.4 wire: 0 on success | (Historical; not relevant for modern kernels.) |
| `-1` + errno | Error. |

The kernel itself follows the modern convention since Linux 2.6.x: return the new nice value (range -20..19) on success, or a negative errno.

### errors

| Errno | Trigger |
|---|---|
| `EPERM` | Negative inc requested without CAP_SYS_NICE / RLIMIT_NICE; new nice would fall below the permitted floor. |
| `EACCES` | LSM denial (security_task_setnice). |

`nice` cannot return -EINVAL because inc is clamped via the new-nice clamp; very negative inc just hits the EPERM gate.

### abi surface

```text
__NR_nice (x86_64) = 34
__NR_nice (i386)   = 34
__NR_nice (arm64)  = NOT EXPOSED  (deprecated by generic-syscall; libc emulates via setpriority)
__NR_nice (generic) = NOT EXPOSED

NICE_MIN     = -20
NICE_MAX     =  19
```

### compatibility contract

REQ-1: Syscall number is **34** on x86_64 and i386. NOT exposed on arm64 / RISC-V / generic; libc must emulate via `setpriority`.

REQ-2: `nice(inc)` is exactly equivalent to:
```
int new_nice = clamp(task_nice(current) + inc, NICE_MIN, NICE_MAX);
return setpriority(PRIO_PROCESS, 0, new_nice);  // returns 0 + new_nice via wire
```

REQ-3: The new nice value is clamped to [-20, 19] before any permission check; the permission check uses the clamped value as the proposed new value.

REQ-4: Permission gate identical to setpriority's per-task gate:
- Lowering nice (new < current) requires CAP_SYS_NICE OR new ≥ (NICE_MAX - RLIMIT_NICE + 1).
- Raising nice (new > current) is free for the owner.

REQ-5: LSM hook `security_task_setnice(current, new_nice)` fires.

REQ-6: Return value on success is the **new** effective nice value (after clamp). Kernel wire returns the value directly (no +20 offset that getpriority uses — this is a known UAPI inconsistency).

REQ-7: For SCHED_FIFO/RR/DEADLINE tasks, nice updates the nominal nice but does NOT affect rt_priority. CFS scheduling parameters are unaffected since the task is not on the CFS queue.

REQ-8: The change is atomic under task_rq_lock: dequeue → static_prio update → set_load_weight → enqueue.

REQ-9: `nice(0)` is a permitted no-op: returns current nice without any state change (but still invokes LSM hook).

REQ-10: Repeated `nice(1)` calls saturate at +19 and return 19 with success (no EPERM since raising nice is always permitted for owner).

REQ-11: A single `nice(-40)` call clamps to -20 and either succeeds (with CAP_SYS_NICE / sufficient RLIMIT_NICE) or returns -EPERM.

REQ-12: The call is async-signal-safe in the kernel's view (no blocking, no allocations).

REQ-13: `nice` does not consult the SCHED_RESET_ON_FORK flag.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `new_nice_clamped` | INVARIANT | per-call: new nice ∈ [-20, 19] before any state change. |
| `cap_or_rlimit_for_lowering` | INVARIANT | per-call: lowering nice ⟹ CAP_SYS_NICE or RLIMIT_NICE satisfied. |
| `rq_lock_held` | INVARIANT | per-set_user_nice: rq lock held over dequeue→reweight→enqueue. |
| `lsm_hook_invoked` | INVARIANT | per-call: security_task_setnice called pre-commit. |
| `wire_returns_new_nice` | INVARIANT | per-call: success ⟹ ret == new effective nice. |
| `saturating_add` | INVARIANT | per-call: cur + inc cannot overflow (saturating arithmetic). |
| `sandbox_floor_respected` | INVARIANT | per-call: sandboxed task: ret ≥ floor. |

### Layer 2: TLA+

`kernel/nice.tla`:
- States: per-task nice + per-task rlimit + per-task sandbox flag.
- Properties:
  - `safety_nice_in_range` — per-task: post-call nice ∈ [-20, 19].
  - `safety_no_partial` — per-call: failure leaves task unchanged.
  - `safety_perm_enforced` — per-call: -EPERM cases never commit.
  - `liveness_terminates` — per-call: O(1).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_call` post: ret ∈ [-20, 19] ∨ ret < 0 (errno) | `Nice::do_call` |
| `do_call` post: success ⟹ task_nice(current) == ret | `Nice::do_call` |
| `sandbox_clamp` post: ret ≥ task.sandbox_nice_floor when sandboxed | `Nice::sandbox_clamp` |

### Layer 4: Verus / Creusot functional

Per-`nice(2)` man-page equivalence. LTP `nice01..05` pass. POSIX.1-2008 (XSI option) semantics verified. Equivalent to `setpriority(PRIO_PROCESS, 0, current_nice + inc)` modulo wire-ABI differences.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`nice(2)` reinforcement:

- **Per-saturating-add** — defense against per-int-overflow on cur+inc.
- **Per-clamp-before-check** — defense against per-out-of-range nice slipping past permission gate.
- **Per-RLIMIT_NICE strict** — defense against per-priority-elevation without quota.
- **Per-LSM hook pre-commit** — defense against per-policy bypass.
- **Per-rq-lock atomic re-weight** — defense against per-half-weighted task observed by load balancer.

### grsecurity / pax-style reinforcement

- **PaX UDEREF (n/a for this call)** — nice takes no userspace pointers; included for completeness.
- **CAP_SYS_NICE strict for nice decrease** — grsec retains the upstream rule: lowering nice (priority elevation) requires CAP_SYS_NICE or RLIMIT_NICE. Under GRKERNSEC_HARDEN, RLIMIT_NICE alone is insufficient — CAP_SYS_NICE is required.
- **GRKERNSEC_RESLOG on rlimit violations** — every -EPERM from RLIMIT_NICE ceiling is logged with task name/uid/current/attempted nice.
- **Sched_deadline attack-surface reduction** — nice on a SCHED_DEADLINE task is permitted (updates nominal nice only) but audited under sandbox so RT/deadline reconnaissance is observable.
- **PAX_RANDKSTACK at syscall entry** — randomizes kernel stack offset per call; nice is a hot path, so RANDKSTACK uses a low-entropy fast path.
- **No_new_privs gate** — when current has PR_SET_NO_NEW_PRIVS, nice(inc<0) is denied with -EPERM regardless of capability or RLIMIT_NICE. nice(inc>=0) is permitted.
- **Nice clamping under sandbox** — under sandbox flags (PR_SET_NO_NEW_PRIVS, seccomp filter), the effective new nice is clamped to [SANDBOX_NICE_MIN, SANDBOX_NICE_MAX] (e.g., [0, 19]). nice(-5) from a sandboxed nice-0 task returns 0 (clamped to floor) rather than -5 or -EPERM, ensuring forward progress without privilege escalation.
- **GRKERNSEC_CHROOT_NICE** — inside a grsec chroot, lowering nice below 0 requires CAP_SYS_NICE even with RLIMIT_NICE permitting it.
- **KEEPCAPS aware** — capability check honors effective set; SECBIT_KEEP_CAPS does not bypass.
- **Saturating arithmetic** — cur + inc uses saturating_add so an inc of INT_MAX cannot wrap around to a low negative number bypassing the nice-decrease check.
- **Audit on extreme nice** — nice(NICE_MIN) (i.e., elevating to -20) is logged under GRKERNSEC_AUDIT_NICE_DECREASE even on success, to flag potential RT-fingerprinting probes.

