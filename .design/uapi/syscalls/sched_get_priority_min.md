# Tier-5 syscall: sched_get_priority_min(2) — syscall 147

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sched/syscalls.c (SYSCALL_DEFINE1(sched_get_priority_min))
  - include/uapi/linux/sched.h (SCHED_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (147 common sched_get_priority_min)
  - man sched_get_priority_min(2)
-->

## Summary

`sched_get_priority_min(2)` returns the **minimum valid `sched_priority`** for a given scheduling policy. Like its companion `sched_get_priority_max(2)`, it is a pure-function lookup against a fixed kernel constant table: SCHED_FIFO and SCHED_RR yield `1` (the lowest user-visible RT priority — internal RT priority `0` is reserved for non-RT scheduling), and all other policies (SCHED_NORMAL / SCHED_BATCH / SCHED_IDLE / SCHED_DEADLINE) yield `0`. The pair `[sched_get_priority_min(policy), sched_get_priority_max(policy)]` defines the inclusive bound on the `sched_priority` field that may be passed to `sched_setscheduler(2)` / `sched_setparam(2)` for that policy. Critical for: portable RT applications that need to clamp into the kernel's RT range without hardcoding `1` (FreeBSD's lower bound differs); audio / industrial libraries computing relative priority offsets from the minimum; static-analysis tools that verify RT-priority literals are in-range.

This Tier-5 covers the userspace ABI of syscall 147.

## Signature

```c
int sched_get_priority_min(int policy);
```

Rust ABI shim:

```rust
pub fn sys_sched_get_priority_min(policy: i32) -> isize;
```

Syscall number: **147** (x86_64). Generic syscall table: **126**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `policy` | `i32` | IN | One of SCHED_NORMAL, SCHED_FIFO, SCHED_RR, SCHED_BATCH, SCHED_IDLE, SCHED_DEADLINE. SCHED_RESET_ON_FORK bit is NOT permitted (raw policy expected). |

## Return

- **Success**: non-negative integer = minimum priority for the policy.
- **Failure**: `-1` and `errno`.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `policy` is not one of the recognised SCHED_* constants, or contains `SCHED_RESET_ON_FORK`. |

## ABI surface

```text
__NR_sched_get_priority_min (x86_64)  = 147
__NR_sched_get_priority_min (i386)    = 160
__NR_sched_get_priority_min (arm64)   = 126  (generic-syscall)
__NR_sched_get_priority_min (generic) = 126

/* Return mapping */
SCHED_FIFO        -> 1
SCHED_RR          -> 1
SCHED_NORMAL      -> 0
SCHED_BATCH       -> 0
SCHED_IDLE        -> 0
SCHED_DEADLINE    -> 0
```

## Compatibility contract

REQ-1: Syscall number is **147** on x86_64, **126** on arm64/generic. ABI-stable since 2.0.

REQ-2: For SCHED_FIFO and SCHED_RR: return `1`. (Kernel-internal RT priority 0 is the "non-RT" sentinel; users get 1..99.)

REQ-3: For SCHED_NORMAL, SCHED_BATCH, SCHED_IDLE, SCHED_DEADLINE: return `0`.

REQ-4: Unknown `policy` ⟹ `-EINVAL`. The check is exhaustive: `SCHED_RESET_ON_FORK | SCHED_FIFO` falls through to `default`.

REQ-5: Pure function: no per-task lookup; no `pi_lock`; no `rq_lock`.

REQ-6: Reentrant; can be called from any context.

REQ-7: ABI-stable: return value `1` for FIFO/RR is locked into the user-kernel contract.

REQ-8: No capability check.

REQ-9: No pid-ns / user-ns interaction.

REQ-10: Companion identity: for any valid policy `p`,
`sched_get_priority_min(p) ≤ sched_get_priority_max(p)`.

## Acceptance Criteria

- [ ] AC-1: `sched_get_priority_min(SCHED_FIFO)` returns `1`.
- [ ] AC-2: `sched_get_priority_min(SCHED_RR)` returns `1`.
- [ ] AC-3: `sched_get_priority_min(SCHED_NORMAL)` returns `0`.
- [ ] AC-4: `sched_get_priority_min(SCHED_BATCH)` returns `0`.
- [ ] AC-5: `sched_get_priority_min(SCHED_IDLE)` returns `0`.
- [ ] AC-6: `sched_get_priority_min(SCHED_DEADLINE)` returns `0`.
- [ ] AC-7: `sched_get_priority_min(99)` (bogus) returns `-EINVAL`.
- [ ] AC-8: `sched_get_priority_min(-1)` returns `-EINVAL`.
- [ ] AC-9: `sched_get_priority_min(SCHED_FIFO | SCHED_RESET_ON_FORK)` returns `-EINVAL`.
- [ ] AC-10: For all valid policies: `sched_get_priority_min(p) ≤ sched_get_priority_max(p)`.

## Architecture

Rookery surface in `kernel/sched/syscalls.rs`:

```rust
pub fn sys_sched_get_priority_min(policy: i32) -> isize {
    match policy {
        SCHED_FIFO | SCHED_RR => 1,
        SCHED_NORMAL | SCHED_BATCH | SCHED_IDLE | SCHED_DEADLINE => 0,
        _ => -EINVAL as isize,
    }
}
```

The constant `1` is hard-coded rather than expressed as `MIN_USER_RT_PRIO`, because POSIX defines this lower bound for RT classes and Linux freezes it at 1 since 2.0. There is no kernel-side knob to change it.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pure_function` | INVARIANT | call is pure; no global state mutation. |
| `range_constant` | INVARIANT | output ∈ {-EINVAL, 0, 1}. |
| `min_le_max` | INVARIANT | for all valid p: `min(p) ≤ max(p)`. |
| `unknown_policy_rejected` | INVARIANT | policy ∉ valid set ⟹ `-EINVAL`. |
| `reset_on_fork_in_policy_rejected` | INVARIANT | `SCHED_RESET_ON_FORK` bit set ⟹ `-EINVAL`. |

### Layer 2: TLA+

`uapi/sched-get-priority-min.tla`:
- Pure function.
- Properties:
  - `safety_fifo_rr_returns_1` — SCHED_FIFO/RR ⟹ 1.
  - `safety_others_return_0` — SCHED_NORMAL/BATCH/IDLE/DEADLINE ⟹ 0.
  - `safety_unknown_returns_einval` — unknown policy ⟹ -EINVAL.
  - `safety_companion_invariant` — pair with `sched_get_priority_max`: min ≤ max for every valid policy.
  - `liveness_terminates` — call returns synchronously.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_sched_get_priority_min` post: output matches static policy table | `sys_sched_get_priority_min` |
| `sys_sched_get_priority_min` ∧ `sys_sched_get_priority_max` joint post: min ≤ max | pair |

### Layer 4: Verus/Creusot functional

Per-`sched_get_priority_min(2)` man page; per-LTP `testcases/kernel/syscalls/sched_get_priority_min/*`; per-glibc `sysdeps/unix/sysv/linux/sched_get_priority_min.c`; per-POSIX.1-2008.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

sched_get_priority_min reinforcement:

- **Per-pure-function: no state to corrupt** — defense against per-syscall-induced state mutation.
- **Per-policy switch exhaustive** — defense against per-fall-through into RT path for non-RT policy.
- **Per-pair-with-max companion invariant `min ≤ max`** — invariant enforced; defense against per-table-corruption that produces inconsistent bounds.
- **Per-1-is-ABI-frozen** — defense against per-runtime-mutation that would silently change the ABI-locked return for SCHED_FIFO/RR; the value of 1 has been stable since 2.0.
- **Per-internal-RT-priority-0 reserved as non-RT sentinel** — defense against per-class-aliasing where userspace 0 would collide with kernel-internal "no RT".

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF irrelevant** — `sched_get_priority_min` takes a scalar `int`; no userspace pointer dereference.
- **PAX_RANDKSTACK on every entry** — randomise kstack on every entry; defense against per-syscall-entry kstack-spray.
- **GRKERNSEC_HIDESYM on RT-priority constants** — defense against per-recon of the internal RT bound layout.
- **Deprecated-syscall ENOSYS gate (off by default)** — grsec hardened policy may force `sched_get_priority_min` to `-ENOSYS`, forcing applications onto hardcoded constants (the value of 1 has been ABI-stable for decades).
- **Per-uid rate-limit** — call rate above 10k/s per uid audit-logged.
- **Audit cross-policy probe** — repeated `sched_get_priority_min` calls iterating across the SCHED_* range audit-logged as scheduler-class reconnaissance.
- **Constant-time return** — same cycle count on success and `-EINVAL` paths to defeat timing-side-channel probing.
- **Refuse SCHED_DEADLINE on non-deadline-capable sandbox** — grsec hardened policy may return `-EINVAL` for SCHED_DEADLINE if the calling uid lacks deadline scheduling capability.
- **Companion-symmetry enforced** — `sched_get_priority_min` and `sched_get_priority_max` MUST be returned by the same policy-table source; defense against per-rogue-LSM patching one and not the other.
- **GRKERNSEC_PROC_GID gate `/proc/<pid>/sched`** — pair with this syscall for coherent RT-priority observability policy.
- **GRKERNSEC_SECCOMP filter** — sandbox SECCOMP filters may pre-clamp to return `0` for SCHED_FIFO/RR; defense against untrusted workload RT-scheduling probes.
- **CAP_SYS_NICE pre-probe gating (off by default)** — under hardened policy, `sched_get_priority_min(SCHED_FIFO)` returning 1 may be gated on the caller possessing CAP_SYS_NICE; defense against per-RT-priority enumeration by unprivileged reconnaissance.
- **Inter-policy table parity** — `sched_get_priority_min` and `sched_get_priority_max` MUST share a single static policy table; defense against per-rogue-LSM patching one syscall and not the other.
- **Per-vDSO not provided** — `sched_get_priority_min` is intentionally NOT exported via vDSO; defense against per-vDSO-divergence from canonical syscall path.
- **Audit suspicious policy values** — `policy < -10` or `policy > 100` audit-logged as a possible argument-fuzzing probe.
- **Companion-test on boot** — boot-time self-test verifies `min(p) ≤ max(p)` for every valid policy; defense against per-table-corruption introduced by a misapplied patch.
- **CAP_SYS_RESOURCE pre-check for cap-raise** — under hardened policy, any future tunable that lifts the lower bound requires CAP_SYS_RESOURCE in init user-ns; defense against per-userns CAP-laundering.
- **Refuse policy enumeration from sandbox** — under SECCOMP-strict, `sched_get_priority_min` for any unsupported policy returns `-EINVAL` even on a recognised constant; defense against per-policy availability probing.
- **GRKERNSEC_RANDPID coarsening unaffected** — `sched_get_priority_min` returns a constant; no PID-derived state leaks via this syscall.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `kernel/sched/types.md` Tier-3: internal RT-priority constants.
- `sched_get_priority_max.md` companion.
- `sched_rr_get_interval.md` sibling.
- `sched_setscheduler.md` / `sched_setattr.md` siblings.
- glibc / musl userspace wrappers.
- Implementation code.
