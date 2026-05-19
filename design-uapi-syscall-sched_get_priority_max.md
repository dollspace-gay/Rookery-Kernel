---
title: "Tier-5 syscall: sched_get_priority_max(2) — syscall 146"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sched_get_priority_max(2)` returns the **maximum valid `sched_priority`** for a given scheduling policy. For Linux this is a pure-function lookup with no per-task or per-system state: SCHED_FIFO and SCHED_RR yield `99` (the POSIX upper bound on RT priorities), and all other policies (SCHED_NORMAL / SCHED_BATCH / SCHED_IDLE / SCHED_DEADLINE) yield `0`. The syscall predates `sched_setattr(2)` and exists so portable code can size its RT priority range against the running kernel without hardcoding `MAX_USER_RT_PRIO - 1`. The companion `sched_get_priority_min(2)` returns the lower bound. Critical for: portability across POSIX.1b RT systems (Linux, FreeBSD, AIX, QNX, Solaris) that historically used different numeric RT-priority bounds; build-time priority constants in audio / industrial libraries; safe RT-priority clamping in scheduler-wrapping libraries (libuv, abseil-cpp, glib).

This Tier-5 covers the userspace ABI of syscall 146. The constant `MAX_USER_RT_PRIO - 1 = 99` is fixed in `include/linux/sched/prio.h`.

### Acceptance Criteria

- [ ] AC-1: `sched_get_priority_max(SCHED_FIFO)` returns `99`.
- [ ] AC-2: `sched_get_priority_max(SCHED_RR)` returns `99`.
- [ ] AC-3: `sched_get_priority_max(SCHED_NORMAL)` returns `0`.
- [ ] AC-4: `sched_get_priority_max(SCHED_BATCH)` returns `0`.
- [ ] AC-5: `sched_get_priority_max(SCHED_IDLE)` returns `0`.
- [ ] AC-6: `sched_get_priority_max(SCHED_DEADLINE)` returns `0`.
- [ ] AC-7: `sched_get_priority_max(99)` (bogus) returns `-EINVAL`.
- [ ] AC-8: `sched_get_priority_max(-1)` returns `-EINVAL`.
- [ ] AC-9: `sched_get_priority_max(SCHED_FIFO | SCHED_RESET_ON_FORK)` returns `-EINVAL` (raw policy expected).
- [ ] AC-10: Call from any task / any uid / any namespace succeeds (no auth gating).

### Architecture

Rookery surface in `kernel/sched/syscalls.rs`:

```rust
pub fn sys_sched_get_priority_max(policy: i32) -> isize {
    match policy {
        SCHED_FIFO | SCHED_RR => (MAX_USER_RT_PRIO - 1) as isize,   /* 99 */
        SCHED_NORMAL | SCHED_BATCH | SCHED_IDLE | SCHED_DEADLINE => 0,
        _ => -EINVAL as isize,
    }
}
```

`MAX_USER_RT_PRIO` is a build-time constant in `kernel/sched/types.rs`:

```rust
pub const MAX_USER_RT_PRIO: i32 = 100;   /* RT priorities 0..99 in kernel, user sees 1..99 */
pub const MAX_RT_PRIO: i32      = MAX_USER_RT_PRIO;
```

User-visible RT range: `[1, 99]` (kernel internally indexes `[0, 99]`, with `0` reserved for non-RT).

### Out of Scope

- `kernel/sched/types.md` Tier-3: scheduler-class internal constants.
- `sched_get_priority_min.md` sibling (lower bound).
- `sched_rr_get_interval.md` sibling (RR time-slice).
- `sched_setscheduler.md` / `sched_setattr.md` siblings.
- glibc / musl userspace wrappers.
- Implementation code.

### signature

```c
int sched_get_priority_max(int policy);
```

Rust ABI shim:

```rust
pub fn sys_sched_get_priority_max(policy: i32) -> isize;
```

Syscall number: **146** (x86_64). Generic syscall table: **125**.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `policy` | `i32` | IN | One of SCHED_NORMAL, SCHED_FIFO, SCHED_RR, SCHED_BATCH, SCHED_IDLE, SCHED_DEADLINE. SCHED_RESET_ON_FORK NOT permitted in this argument (raw policy expected). |

### return

- **Success**: non-negative integer = maximum priority for the policy.
- **Failure**: `-1` and `errno`.

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `policy` is not one of the recognised SCHED_* constants, or contains `SCHED_RESET_ON_FORK` (this syscall expects the raw policy). |

### abi surface

```text
__NR_sched_get_priority_max (x86_64)  = 146
__NR_sched_get_priority_max (i386)    = 159
__NR_sched_get_priority_max (arm64)   = 125  (generic-syscall)
__NR_sched_get_priority_max (generic) = 125

/* Return mapping */
SCHED_FIFO        -> 99
SCHED_RR          -> 99
SCHED_NORMAL      -> 0
SCHED_BATCH       -> 0
SCHED_IDLE        -> 0
SCHED_DEADLINE    -> 0
```

### compatibility contract

REQ-1: Syscall number is **146** on x86_64, **125** on arm64/generic. ABI-stable since 2.0.

REQ-2: For SCHED_FIFO and SCHED_RR: return `MAX_USER_RT_PRIO - 1` which is **99** since 2.6.

REQ-3: For SCHED_NORMAL, SCHED_BATCH, SCHED_IDLE, SCHED_DEADLINE: return `0`.

REQ-4: Unknown `policy` ⟹ `-EINVAL`. The check is `switch(policy) { case SCHED_*: ...; default: -EINVAL; }` so `SCHED_RESET_ON_FORK | SCHED_FIFO` falls into `default`.

REQ-5: Pure function: no per-task lookup, no `pi_lock`, no `rq_lock` taken.

REQ-6: Atomic with respect to nothing; can be called from any context including signal handlers (although the libc wrapper is not async-signal-safe by POSIX standard, the kernel implementation is reentrant).

REQ-7: ABI-stable: the return value `99` is locked into the user-kernel contract; raising `MAX_USER_RT_PRIO` would require coordinated ABI change.

REQ-8: No capability check.

REQ-9: No pid-ns / user-ns interaction (pure constant function).

REQ-10: SCHED_DEADLINE returning `0` is informational; deadline parameters are not expressed through `sched_priority` at all and the value is meaningless for that policy.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pure_function` | INVARIANT | call is pure; no global state mutation. |
| `range_constant` | INVARIANT | output ∈ {-EINVAL, 0, 99}. |
| `unknown_policy_rejected` | INVARIANT | policy ∉ valid set ⟹ `-EINVAL`. |
| `reset_on_fork_in_policy_rejected` | INVARIANT | `SCHED_RESET_ON_FORK` bit set ⟹ `-EINVAL`. |
| `no_lock_acquired` | INVARIANT | no rq_lock / pi_lock taken. |

### Layer 2: TLA+

`uapi/sched-get-priority-max.tla`:
- Pure function; no state.
- Properties:
  - `safety_fifo_rr_returns_99` — SCHED_FIFO/RR ⟹ 99.
  - `safety_others_return_0` — SCHED_NORMAL/BATCH/IDLE/DEADLINE ⟹ 0.
  - `safety_unknown_returns_einval` — unknown policy ⟹ -EINVAL.
  - `liveness_terminates` — call returns synchronously without blocking.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_sched_get_priority_max` post: output matches static policy table | `sys_sched_get_priority_max` |
| `sys_sched_get_priority_max` post: never mutates any kernel state | `sys_sched_get_priority_max` |

### Layer 4: Verus/Creusot functional

Per-`sched_get_priority_max(2)` man page; per-LTP `testcases/kernel/syscalls/sched_get_priority_max/*`; per-glibc `sysdeps/unix/sysv/linux/sched_get_priority_max.c`; per-POSIX.1-2008 specification.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

sched_get_priority_max reinforcement:

- **Per-pure-function: no state to corrupt** — defense against per-syscall-induced state mutation; this syscall is read-only on a static table.
- **Per-policy switch exhaustive** — defense against per-fall-through into RT path for non-RT policy.
- **Per-SCHED_RESET_ON_FORK not accepted** — defense against per-policy-modifier passed where raw policy expected.
- **Per-MAX_USER_RT_PRIO compile-time-frozen** — defense against per-runtime-mutation that would silently change the ABI-locked return for SCHED_FIFO/RR.
- **Per-pair-with-min companion invariant `max ≥ min`** — defense against per-table-corruption producing inconsistent bounds when paired with `sched_get_priority_min`.

### grsecurity/pax-style reinforcement

- **PaX UDEREF irrelevant** — `sched_get_priority_max` takes a scalar `int` argument; no userspace pointer dereference.
- **PAX_RANDKSTACK on every entry** — randomise kstack on every entry even for trivial syscalls; defense against per-syscall-entry-coalescing of kstack layout for spray attacks.
- **GRKERNSEC_HIDESYM on `MAX_USER_RT_PRIO` constant table** — defense against per-recon: while `99` is documented, the underlying table indirection in kernel/sched/types.rs is kept out of `/proc/kallsyms`.
- **Deprecated-syscall ENOSYS gate (off by default)** — grsec hardened policy may force `sched_get_priority_max` to `-ENOSYS`, forcing applications onto hardcoded constants (which is reasonable for an ABI-frozen value of 99); defense against per-legacy-API surface, although this is one of the least dangerous syscalls.
- **Per-uid rate-limit** — call rate above 10k/s per uid audit-logged as a possible busy-loop probe.
- **Audit cross-policy probe** — repeated `sched_get_priority_max` calls iterating across the SCHED_* range from a single process audit-logged as scheduler-class reconnaissance.
- **Cap untrusted user policy enumeration** — under hardened policy, only the policies actually permitted to the calling uid are reported with their max priority; others return `-EINVAL` even if the policy is otherwise valid.
- **GRKERNSEC_SECCOMP filter on RT-policy enumeration** — sandbox SECCOMP filters may pre-clamp `sched_get_priority_max` to return `0` for SCHED_FIFO/RR for untrusted workloads.
- **Refuse policy = SCHED_DEADLINE under non-deadline-capable sandbox** — grsec hardened policy returns `-EINVAL` for SCHED_DEADLINE to a uid that cannot use deadline scheduling.
- **Constant-time return** — even on the error path, the same number of cycles is taken to defeat timing-side-channel probing of which policies are recognised (modest defense, but cheap).
- **GRKERNSEC_PROC_GID gating** — pair with `sched_get_priority_max` to ensure the policy of "RT priorities not observable to non-privileged uid" is coherent across both this syscall and `/proc/<pid>/sched`.
- **CAP_SYS_NICE pre-probe gating (off by default)** — under hardened policy, `sched_get_priority_max(SCHED_FIFO)` returning 99 may be gated on the caller possessing CAP_SYS_NICE; defense against per-RT-priority enumeration by unprivileged reconnaissance.
- **Inter-policy table parity** — `sched_get_priority_max` and `sched_get_priority_min` MUST share a single static policy table; defense against per-rogue-LSM patching one syscall and not the other to confuse priority-clamping libraries.
- **Per-vDSO not provided** — `sched_get_priority_max` is intentionally NOT exported via vDSO; defense against per-vDSO-divergence from canonical syscall path (the constant is so trivial that vDSO acceleration is unnecessary).
- **Audit suspicious policy values** — `policy < -10` or `policy > 100` audit-logged as a possible argument-fuzzing probe.
- **CAP_SYS_RESOURCE pre-check for any future cap-raise to `MAX_USER_RT_PRIO`** — under hardened policy, lifting the constant requires CAP_SYS_RESOURCE in init user-ns; defense against per-userns CAP-laundering for hypothetical future tunables.

