---
title: "Tier-5 syscall: rseq_slice_yield(2) — forward-looking (ENOSYS in 7.1.0-rc2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`rseq_slice_yield(2)` is a **proposed syscall** that, when invoked from inside an active rseq critical section, performs a cooperative scheduler yield while preserving — rather than aborting — the rseq commit intent. The classical rseq design treats any rescheduling as an implicit abort: the kernel rewinds the user IP to the `abort_ip` of the active critical section so userspace can retry. That semantics is correct but expensive when the section is long and the user runtime *wants* to yield (lock-elision contention backoff, latency-fair scheduling) — the abort discards all the work already done in the section and forces re-execution after the yield.

`rseq_slice_yield(2)` lets the user runtime opt into "yield without abort": the kernel records that the critical section is parked, performs the yield, and on resume the user code continues from the syscall return point with the registered cpu-id refreshed. The commit step still runs only if rseq invariants (no migration mid-commit) hold at the actual commit instruction.

The current 7.1.0-rc2 baseline **does not** ship this syscall: `__NR_rseq_slice_yield = 472` is reserved and the entry trampoline returns `-ENOSYS`. Userspace MUST continue to use bare `sched_yield(2)` (which forces an rseq abort) until the syscall is wired in a later phase.

Critical for: tcmalloc / mimalloc / jemalloc per-cpu cache contention yielding without losing the slow-path-prep work; userspace RCU read-side adaptive backoff; cooperative coroutine schedulers with rseq-pinned per-cpu queues.

### Acceptance Criteria

- [ ] AC-1: In 7.1.0-rc2, `syscall(__NR_rseq_slice_yield, 0, 0)` returns `-1` with `errno == ENOSYS`.
- [ ] AC-2: After implementation: unregistered task: `ENOSYS`.
- [ ] AC-3: After implementation: `flags = RSEQ_SLICE_YIELD_F_RESERVED` returns `EINVAL`.
- [ ] AC-4: After implementation: `rseq_cs_offset` mismatch returns `EINVAL`.
- [ ] AC-5: After implementation: inside a critical section, yield does NOT trigger abort_ip; resume continues at syscall+4.
- [ ] AC-6: After implementation: post-yield, `rseq.cpu_id` reflects current CPU.
- [ ] AC-7: After implementation: post-migration commit instruction still aborts (existing rseq abort path).
- [ ] AC-8: After implementation: `AGGRESSIVE` flag forces schedule even when `!need_resched`.
- [ ] AC-9: After implementation: return value `0` (rescheduled) vs `1` (refresh-only) accurately distinguishes.
- [ ] AC-10: After implementation: signal-handler invocation is well-defined.
- [ ] AC-11: After implementation: no-op out-of-section call returns 1, refreshes cpu_id.

### Architecture

```rust
#[syscall(nr = 472, abi = "sysv")]
pub fn sys_rseq_slice_yield(flags: u32, rseq_cs_offset: u32) -> isize {
    /* Forward-looking: not implemented in 7.1.0-rc2. */
    -ENOSYS as isize
}
```

When implemented:

`Rseq::do_slice_yield(flags, rseq_cs_offset) -> isize`:
1. if (flags & RSEQ_SLICE_YIELD_F_RESERVED) != 0 { return -EINVAL; }
2. let task = current();
3. let rseq = task.rseq.as_ref().ok_or(ENOSYS)?;
4. /* Confirm rseq_cs descriptor matches the registered area */
5. let cs_ptr = rseq.read_rseq_cs_field().map_err(|_| EFAULT)?;
6. if cs_ptr.0 != 0 {
7.   if cs_ptr.offset_from(rseq.user_addr) != rseq_cs_offset as i64 {
8.     return -EINVAL;
9.   }
10. } else if rseq_cs_offset != 0 {
11.  return -EINVAL;
12. }
13. /* Mark "park": the next preempt-notifier MUST NOT call rseq_abort. */
14. task.rseq_state.set_parked(true);
15. /* Yield */
16. let rescheduled = if (flags & RSEQ_SLICE_YIELD_F_AGGRESSIVE) != 0 {
17.   schedule(); true
18. } else {
19.   cond_resched()
20. };
21. /* Resume: clear park, refresh cpu_id */
22. task.rseq_state.set_parked(false);
23. rseq.publish_cpu_id(smp_processor_id()).map_err(|_| EFAULT)?;
24. if rescheduled { 0 } else { 1 }
```

The key novelty is the `parked` flag consulted by the existing rseq notifier:

```rust
fn rseq_notify_resume(task) {
    if task.rseq_state.parked() { return; }   // slice-yield: skip abort
    /* existing IP-in-critical-section abort path unchanged */
    if rseq_ip_in_cs(task) { rseq_abort_user_to_abort_ip(task); }
}
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `enosys_in_baseline` | INVARIANT | per-7.1.0-rc2: sys_rseq_slice_yield always returns -ENOSYS. |
| `no_abort_during_park` | INVARIANT | per-implemented: while parked == true, rseq_notify_resume does not call abort. |
| `cpu_id_refresh_post_yield` | INVARIANT | per-implemented: post-yield, rseq.cpu_id == smp_processor_id(). |
| `commit_abort_still_holds` | INVARIANT | per-implemented: post-resume migration ⟹ commit insn aborts via existing path. |
| `flag_reserved_zero` | INVARIANT | per-implemented: any bit in RSEQ_SLICE_YIELD_F_RESERVED ⟹ EINVAL. |
| `no_cap_widen` | INVARIANT | per-implemented: does not change creds, IP, SP. |

### Layer 2: TLA+

`kernel/rseq-slice-yield.tla`:
- States: per-task rseq.cpu_id, per-task parked flag, per-task ip-in-cs predicate, CPU assignment.
- Properties:
  - `safety_enosys_baseline` — in 7.1.0-rc2, every call returns ENOSYS.
  - `safety_no_abort_when_parked` — parked ⟹ no abort_ip rewrite at preempt.
  - `safety_cpu_id_consistent_post_yield` — yield exit ⟹ cpu_id == current CPU.
  - `safety_commit_still_safe` — post-resume commit insn aborts on cpu mismatch (existing rseq invariant intact).
  - `safety_no_caps` — no cred mutation.
  - `liveness_terminates` — every call returns (cond_resched terminates).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_rseq_slice_yield` post (baseline): returns -ENOSYS unconditionally | `sys_rseq_slice_yield` (baseline) |
| `do_slice_yield` post (implemented): success ⟹ rseq.cpu_id == smp_processor_id() | `Rseq::do_slice_yield` |
| `rseq_notify_resume` post: parked ⟹ abort_ip not written | `rseq_notify_resume` |
| `publish_cpu_id` post: rseq.cpu_id_start == rseq.cpu_id (memory-ordered) | `Rseq::publish_cpu_id` |

### Layer 4: Verus / Creusot functional

- Baseline: probe call equals `errno == ENOSYS`.
- Implemented: equivalence to "schedule then refresh cpu_id without abort" specification; selftests in `tools/testing/selftests/rseq/` (yield-fairness variant) pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`rseq_slice_yield(2)` reinforcement (forward-looking):

- **Per-`parked` flag scoped strictly to syscall duration** — defense against per-permanent-park-abuse keeping aborts off across non-syscall reschedules.
- **Per-`rseq_cs_offset` match check** — defense against per-mis-coordinated descriptor; userspace cannot signal "I'm in section X" when it's actually in Y.
- **Per-`flags` reserved-bit-zero** — defense against per-extension-field smuggling.
- **Per-`cpu_id` publish via existing notifier path** — defense against per-stale-cpu-id ordering bugs.
- **Per-commit-path abort intact** — defense against per-rseq invariant violation (migration mid-commit is still aborted).
- **Per-no cap / cred change** — defense against per-priv-escalation via novel syscall.
- **Per-`-ENOSYS` baseline semantics** — defense against per-EINVAL libc-probe confusion.

## Grsecurity / PaX surface

- **GRKERNSEC_RSEQ_HARDEN** — grsec adds a global toggle to disable rseq entirely. When set, both `rseq(2)` and `rseq_slice_yield(2)` return `ENOSYS` regardless of build config. Defends against rseq-mediated TOCTOU window manipulation (some side-channel attacks abuse rseq abort+retry as a precise timing oracle).
- **PaX UDEREF on `rseq.rseq_cs` field read** — the per-syscall validation that re-reads the user-side rseq_cs pointer goes through SMAP/UDEREF; a forged kernel-range address in the field is rejected.
- **PAX_USERCOPY_HARDEN on rseq area** — fixed-size copies into the per-task rseq struct use whitelisted slab.
- **PAX_RANDKSTACK at entry** — the syscall entry randomizes kernel stack offset to defeat any rseq-as-stack-probe attempts.
- **GRKERNSEC_PROC_USERGROUP neutral** — rseq state is not in `/proc`; no info leak surface.
- **PaX KSPP no_new_privs neutral** — slice-yield does not exec, does not change creds.
- **Anti-fingerprint** — return value is binary (0 / 1); no scheduler-internal data leaked.
- **GRKERNSEC_AUDIT_RSEQ (planned)** — when the syscall lands, an audit record `AUDIT_RSEQ_SLICE_YIELD` would log task / cpu / aggressive-flag; useful for diagnosing pathologic yield-loops.
- **PaX REFCOUNT neutral** — no refcounts manipulated.
- **Per-`-ENOSYS` is the only baseline outcome** — slot 472 sealed to `sys_ni_syscall` thunk under grsec until the implementation lands.

## Open Questions

- Q1: Should `parked` set across signal-handler entry abort the section anyway (signal handlers ARE valid abort triggers in classical rseq)? Lean yes — signal entry clears `parked` and resumes normal abort semantics. Defer.
- Q2: Distinguish "rescheduled with migration" (return 0) from "rescheduled same-CPU" (return 2)? Could be useful; defer.
- Q3: A multi-section variant where userspace passes a list of valid rseq_cs offsets (cooperative scheduler over multiple lock-elision sites)? Out of scope for v1.
- Q4: Interaction with `sched_setattr(SCHED_DEADLINE)` — should aggressive yield be denied on deadline tasks? TBD.

## Out of Scope

- `rseq(2)` registration syscall (Tier-5 separate doc — `rseq.md`).
- Existing rseq abort/notifier semantics (Tier-3 in `kernel/rseq.md`).
- `sched_yield(2)` (Tier-5 separate doc — bare yield with abort).
- Per-cpu allocator integration patterns (out of scope at UAPI level).
- Implementation code.

### signature

```c
int rseq_slice_yield(uint32_t flags, uint32_t rseq_cs_offset);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `flags` | `uint32_t` | in | Bitmask: `RSEQ_SLICE_YIELD_F_AGGRESSIVE` (request schedule even if remaining slice large), reserved otherwise. |
| `rseq_cs_offset` | `uint32_t` | in | Offset (within registered rseq struct) of the rseq_cs descriptor whose section the caller is yielding from. Used as a sanity check; the kernel verifies the descriptor matches `rseq.rseq_cs`. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; yielded and rescheduled; caller's `rseq.cpu_id` updated on resume. |
| `1` | Success; no reschedule occurred (no eligible candidate) but `rseq.cpu_id` refreshed. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `ENOSYS` | **Always, in 7.1.0-rc2.** The syscall is reserved but not implemented. |
| `ENOSYS` | (post-implementation) Caller has no registered rseq area. |
| `EINVAL` | Unknown `flags` bits; `rseq_cs_offset` does not match `current->rseq->rseq_cs`. |
| `EFAULT` | The registered rseq area is unreadable or unaligned. |
| `EPERM` | (reserved) seccomp/LSM-imposed denial. |

### abi surface

```text
__NR_rseq_slice_yield (x86_64)   = 472   /* reserved; ENOSYS in 7.1.0-rc2 */
__NR_rseq_slice_yield (arm64)    = 472
__NR_rseq_slice_yield (generic)  = 472

#define RSEQ_SLICE_YIELD_F_AGGRESSIVE   0x00000001
#define RSEQ_SLICE_YIELD_F_RESERVED     0xFFFFFFFE   /* any bit ⟹ EINVAL */
```

### compatibility contract

REQ-1: In 7.1.0-rc2, `rseq_slice_yield` is wired into the syscall table at slot 472 but the implementation body returns `-ENOSYS`. Probing returns `ENOSYS`, not `EINVAL`.

REQ-2: `__NR_rseq_slice_yield = 472` slot is reserved cross-arch.

REQ-3: When implemented (target: phase-E or later), the syscall MUST:
  - Validate `flags & RSEQ_SLICE_YIELD_F_RESERVED == 0`, else `EINVAL`.
  - Confirm `current->rseq != NULL`, else `ENOSYS` (matches behavior of other rseq operations on unregistered tasks).
  - Read `current->rseq->rseq_cs` and confirm `rseq_cs_offset` matches the offset of that descriptor within the rseq area, else `EINVAL`.
  - Perform a `schedule()` (or `cond_resched()` with `AGGRESSIVE`) WITHOUT triggering the rseq abort path.

REQ-4: The defining novelty: when the scheduler is entered from this syscall (as opposed to from a preemption-time IP-in-critical-section detection), the abort handler MUST NOT be invoked. The rseq layer treats this entry as "parked": the rseq_cs intent remains live; only the registered `cpu_id_start` / `cpu_id` fields are refreshed on resume.

REQ-5: If, between the yield and resume, the task is migrated to another CPU, the commit instruction will (per existing rseq semantics) detect the cpu mismatch via `cpu_id_start != cpu_id` and abort to the user-supplied `abort_ip`. The slice-yield does NOT bypass this safety net — it only avoids forcing the abort at yield time.

REQ-6: Concurrency: the kernel-side update of `cpu_id_start` / `cpu_id` is published under the existing rseq notifier path; readers in userspace see ordered updates per the rseq memory model.

REQ-7: `RSEQ_SLICE_YIELD_F_AGGRESSIVE` semantics: if clear, the kernel performs `cond_resched()` (cheap path; yield only if `need_resched`); if set, the kernel performs an unconditional `schedule()` (give up the remainder of the slice unconditionally).

REQ-8: No capability or LSM hook is added; the syscall does not widen any privilege.

REQ-9: `seccomp` may filter the syscall by number (472) like any other.

REQ-10: Backward compat: kernels without `rseq_slice_yield` MUST return `-ENOSYS` from slot 472; userspace runtimes detect via probe and fall back to `sched_yield(2)` + abort-retry loop.

REQ-11: Forward compat: future flags MAY extend behavior; the reserved-bit-zero rule preserves headroom.

REQ-12: The syscall is async-signal-safe in the sense that running it inside a signal handler is well-defined (it simply yields once); it is NOT a fast-path syscall — costs are dominated by the schedule call itself.

REQ-13: The syscall does NOT touch the registered `rseq.flags`, `rseq.abort_ip`, or any user-controlled IP/SP register state.

REQ-14: The return value distinguishes "yielded" (0) from "no candidate found, refresh-only" (1); userspace libraries can use this signal to back off (or not).

REQ-15: The syscall MUST be a no-op (return 1, refresh cpu_id) when called outside any active rseq critical section, but only if `current->rseq` is registered. This matches a useful "rate-limit refresh" idiom.

