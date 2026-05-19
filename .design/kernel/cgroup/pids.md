# Tier-3: kernel/cgroup/pids.c — pids controller (max process count + per-cgroup fork-charging hierarchy)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/cgroup/cgroup-core.md
upstream-paths:
  - kernel/cgroup/pids.c
  - include/linux/cgroup.h
  - Documentation/admin-guide/cgroup-v2.rst (pids section)
-->

## Summary

The pids controller (cgroup-v2 / v1) limits the number of processes (tasks/PIDs) a cgroup may contain — a critical anti-fork-bomb defense + per-tenant resource isolation primitive. Each cgroup has `pids.max` (limit), `pids.current` (running count), `pids.events` (max-hit-counter). On `fork(2)`, kernel charges +1 to the destination cgroup's count + walks ancestor chain charging each level; if any level exceeds limit, the fork fails with -EAGAIN. On task exit, count decremented per ancestor walk. Uses per-CPU counter for hot-path scaling.

This Tier-3 covers `kernel/cgroup/pids.c` (~460 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pids_cgroup` | per-cgroup pids state | `kernel::cgroup::pids::PidsCgroup` |
| `pids_css_alloc(parent_css)` | per-cgroup alloc | `PidsCgroup::css_alloc` |
| `pids_css_free(css)` | per-cgroup free | `PidsCgroup::css_free` |
| `pids_can_attach(css, tset)` | pre-attach check | `PidsCgroup::can_attach` |
| `pids_attach(css, tset)` | post-attach charge transfer | `PidsCgroup::attach` |
| `pids_can_fork(task, css)` | pre-fork charge | `PidsCgroup::can_fork` |
| `pids_cancel_fork(task, css)` | rollback on fork-fail | `PidsCgroup::cancel_fork` |
| `pids_release(task)` | post-task-exit decrement | `PidsCgroup::release` |
| `pids_max_show` / `pids_max_write` | pids.max R/W | `PidsCgroup::max` |
| `pids_current_read` | pids.current R | `PidsCgroup::current` |
| `pids_events_show` | pids.events R | `PidsCgroup::events` |
| `pids_charge(pids, num)` | charge + propagate up | `PidsCgroup::charge` |
| `pids_uncharge(pids, num)` | uncharge + propagate up | `PidsCgroup::uncharge` |
| `pids_try_charge(pids, num)` | charge with limit-check | `PidsCgroup::try_charge` |

## Compatibility contract

REQ-1: Per-cgroup `pids_cgroup`:
- `counter` (atomic64_t; current pids count).
- `limit` (atomic64_t; max allowed; -1 = unlimited).
- `events_local[NR_PIDCG_EVENTS]` (per-event counter array).
- `events_file` (cgroup_file for pids.events tracking).

REQ-2: pids.max format:
- "max" or integer N.
- Atomic64 to allow lockless write+read.

REQ-3: pids.current format:
- Read-only integer.
- Reflects sum of all charges from descendants + this cgroup's tasks.

REQ-4: pids.events format:
- key/value pairs:
  - "max" - count of times this cgroup hit its max limit (per Documentation).

REQ-5: can_fork hook (pre-fork validation):
1. Get target cgroup (current cgroup of forking task).
2. Walk ancestor chain (target → root):
   - Per-cgroup: try_charge(pids, 1):
     - If counter+1 > limit: increment events_local[MAX]; return -EAGAIN.
     - Atomic-CAS counter += 1.
3. If any level fails: rollback prior level charges via cancel_fork.
4. Otherwise: proceed with fork.

REQ-6: cancel_fork hook (rollback):
- Walk ancestors; per-level decrement counter.

REQ-7: release hook (per-task exit):
- Walk ancestors; per-level decrement counter.

REQ-8: can_attach hook (cgroup-migrate validation):
1. Compute net charge: count of moving-tasks - count already in dest.
2. Walk dest ancestors:
   - try_charge(pids, count) — if exceeds: return -EAGAIN.

REQ-9: attach hook (post-cgroup-migrate):
- Source cgroup ancestors decremented by count.
- Dest cgroup ancestors incremented by count.

REQ-10: Per-CPU counter optimization:
- Charge/uncharge use atomic64 for simplicity; for hot-path could use percpu_counter.
- Current Linux uses atomic64_t directly; works at scale due to short-hold + cache-coherence.

REQ-11: Per-cgroup events_local counters:
- Per-event aggregated to events_file when read.
- Per-cgroup-fork-fail: events_local[MAX]++.

REQ-12: PIDs limit interaction with global pid_max:
- Global pid_max is system-wide; pids.max is per-cgroup.
- Global limit checked first; cgroup limit second.

REQ-13: cgroup-v1 mount semantics:
- /sys/fs/cgroup/pids per-controller mount.
- Per-cgroup file pids.max + pids.current.

REQ-14: cgroup-v2 unified mount semantics:
- /sys/fs/cgroup unified hierarchy.
- Per-cgroup pids.max + pids.current + pids.events.
- pids controller toggled via cgroup.subtree_control.

## Acceptance Criteria

- [ ] AC-1: pids.max=10 in cgroup C; fork 11th task; -EAGAIN returned.
- [ ] AC-2: pids.current accurate after 100-task spawn + 50-task exit; reflects 50.
- [ ] AC-3: Hierarchical: parent.pids.max=20; child1.pids.max=15; child2.pids.max=15; child1+child2 combined > 20 → parent rejects 21st fork.
- [ ] AC-4: pids.events: per-fork-fail bumps "max" counter.
- [ ] AC-5: cgroup-migrate: 5 tasks moved from src to dst with dst.pids.max=4 → can_attach returns -EAGAIN.
- [ ] AC-6: pids.max=max: unlimited; verifies large fork count succeeds.
- [ ] AC-7: cgroup-rmdir: empty cgroup released; pids state freed.
- [ ] AC-8: Linux Test Project pids controller suite passes.

## Architecture

`PidsCgroup`:

```
struct PidsCgroup {
  css: CgroupSubsysState,                     // base cgroup_subsys_state
  counter: AtomicI64,
  limit: AtomicI64,                            // -1 = unlimited
  watermark: AtomicI64,                        // high-water mark
  events_local: [AtomicU64; NR_PIDCG_EVENTS],
  events_file: KArc<CgroupFile>,
}
```

`PidsCgroup::can_fork(task, css)`:
1. pids := css_to_pids(css).
2. Walk ancestor chain p ∈ {pids, pids.parent, ..., root}:
   - p.try_charge(1):
     - cur := atomic64_inc_return(&p.counter).
     - If p.limit != -1 && cur > p.limit:
       - atomic64_dec(&p.counter).
       - p.events_local[PIDCG_EVENT_MAX]++.
       - cgroup_file_notify(&p.events_file).
       - Return -EAGAIN.
3. Return 0.

`PidsCgroup::cancel_fork(task, css)`:
1. pids := css_to_pids(css).
2. Walk ancestor chain p:
   - atomic64_dec(&p.counter).

`PidsCgroup::release(task)`:
1. pids := task.cgroups.subsys[pids_subsys_id].
2. Walk ancestor chain p:
   - atomic64_dec(&p.counter).

`PidsCgroup::can_attach(css, tset)`:
1. dest_pids := css_to_pids(css).
2. nr_tasks := tset.count_tasks().
3. Walk dest_pids ancestor chain p:
   - p.try_charge(nr_tasks):
     - cur := atomic64_add_return(nr_tasks, &p.counter).
     - If cur > p.limit && p.limit != -1:
       - atomic64_sub(nr_tasks, &p.counter).
       - return -EAGAIN.
4. Return 0.

`PidsCgroup::attach(css, tset)`:
1. Per-task in tset:
   - src_pids := old cgroup; dst_pids := new cgroup.
   - Walk src ancestor chain (down to common-ancestor): atomic64_dec(&p.counter).
   - Walk dst ancestor chain (down to common-ancestor): atomic64_inc(&p.counter).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `counter_no_underflow` | INVARIANT | per-cgroup counter ≥ 0; defense against double-uncharge wrap. |
| `try_charge_atomic` | INVARIANT | charge succeeds atomically (counter incremented by exactly nr-tasks); rollback on limit-exceed. |
| `ancestor_walk_terminates` | INVARIANT | walk bounded by cgroup tree depth (typically ≤ 8). |
| `events_counter_no_underflow` | INVARIANT | events_local counters ≥ 0; monotonic-only-add. |

### Layer 2: TLA+

`kernel/cgroup/pids_charge.tla`:
- Per-cgroup state: counter ∈ [0, limit].
- Transitions: charge(N) increments by N if cur+N ≤ limit; uncharge(N) decrements by N.
- Properties:
  - `safety_counter_within_limit` — counter ≤ limit at all times (limit may be -1 = unbounded).
  - `safety_atomic_propagate` — multi-level charge atomic: either all levels incremented or all rolled back.
  - `liveness_eventual_release` — after task exit, counter decrements happen.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `PidsCgroup::can_fork` post: returned 0 iff all ancestors charged; returned -EAGAIN iff any level rolled-back | `PidsCgroup::can_fork` |
| `PidsCgroup::release` post: per-ancestor counter decremented exactly once per task | `PidsCgroup::release` |
| `PidsCgroup::can_attach` post: nr_tasks reserved in dst ancestors atomically | `PidsCgroup::can_attach` |
| Per-cgroup events_local[MAX] incremented on every fork-fail | `PidsCgroup::can_fork` |

### Layer 4: Verus/Creusot functional

`Per-fork: ancestor charge succeeds iff every ancestor.counter < ancestor.limit at fork-time`: per-(task, ancestor-chain) the can_fork outcome matches what would result from sequential per-level limit-check.

## Hardening

(Inherits row-1 features from `kernel/cgroup/00-overview.md` § Hardening.)

pids-controller-specific reinforcement:

- **Per-charge atomic with rollback** — defense against partial-charge state leaving counter inconsistent.
- **Counter underflow check (≥ 0)** — defense against double-uncharge wrap causing limit-bypass.
- **Per-cgroup limit set-once via root authority** — defense against unprivileged limit-bypass.
- **Ancestor walk depth bounded** — defense against pathological cgroup-tree depth attack.
- **Events counter monotonic-only** — defense against event-count rollback hiding fork-fail.
- **Per-fork can_fork called before pid alloc** — defense against partial-fork side-effects (PID assigned but cgroup-charge failed).
- **cancel_fork rollback completeness** — defense against partial-rollback on multi-level fail leaving counters inconsistent.
- **Per-attach can_attach + attach paired** — defense against partial-migrate leaving src counter wrong.
- **pids.max="max" interpretation** — defense against integer-overflow on max-int limit values.
- **cgroup_file_notify on max-hit** — defense against silent fork-fail without admin awareness.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `pids.max` write handler copies attacker-supplied "max"/"<int>" string through `kstrtoll`; whitelist `PAGE_SIZE - 1` and reject any oversized write before parse.
- **PAX_KERNEXEC** — pids cftype indirect-call vectors live in `.rodata`; refuse rwx so a kernel-write primitive cannot rewrite `pids_can_attach` to skip the per-cgroup quota check.
- **PAX_RANDKSTACK** — `pids_try_charge` / `pids_can_fork` fire from `fork(2)` / `clone(2)` at attacker-driven depth; randomized kstack offset disrupts ROP through the recursive `parent_pids` walk.
- **PAX_REFCOUNT** — `pids_cgroup->counter` (page_counter family) and `pids_cgroup->css.refcnt` are refcount_t with saturating overflow; defense against fork-storm underflow racing `pids_cancel_attach` against in-flight charge.
- **PAX_MEMORY_SANITIZE** — `pids_css_free` must zero the per-cgroup counter and `events` buffer before kfree; defense against post-free counter residual exposing fork telemetry to a fresh cgroup.
- **PAX_UDEREF** — `pids.max` write keeps SMAP/PAN engaged across `kernfs_ops->write` → `pids_max_write` → `page_counter_set_max`.
- **PAX_RAP/kCFI** — `pids_cgrp_subsys.*` (`css_alloc`, `can_attach`, `cancel_attach`, `attach`, `can_fork`, `cancel_fork`) indirect calls must verify kCFI tag.
- **GRKERNSEC_HIDESYM** — `pids.current` / `pids.max` / `pids.events` readers expose counter values only; refuse to leak the `struct pids_cgroup *` address through any error path or fdinfo.
- **GRKERNSEC_DMESG** — `WARN_ON` on `pids_cancel_attach` rollback failure must not splat raw cgroup / task pointers into dmesg readable by non-CAP_SYSLOG.
- **Per-cgroup process-quota CAP_SYS_RESOURCE** — `pids.max` write must enforce `ns_capable(user_ns, CAP_SYS_RESOURCE)` against the userns owning the cgroup root, not init; defense against per-userns pids granting unbounded fork beyond operator policy.
- **`pids.max="max"` integer-overflow trap** — string "max" must map to `PIDS_MAX_LIMIT` (LLONG_MAX) with an explicit branch, not through `kstrtoll` saturation; defense against attacker passing 2^63-1 to circumvent quota.
- **Per-attach `can_attach + attach` paired** — partial-migration leaving src counter wrong is mitigated upstream; harden by adding rqspinlock timeout WARN so an attacker holding the migration cannot stall the attach indefinitely.
- **`cgroup_file_notify` on max-hit** — already mitigated upstream; harden with rate-limit so a fork-bomb cannot generate event storm on `pids.events`.
- **Host-wide ceiling** — Rookery imposes a sysctl-configured host-wide upper bound on the sum of `pids.max` so a CAP_SYS_RESOURCE-in-container cannot reserve >host-`pid_max` and starve sibling containers.
- **Rationale** — pids cgroup is the kernel's primary defense against fork-bomb DoS in multi-tenant environments; mis-quota grants a container the ability to exhaust host PIDs and lock out admin sessions. The grsec regime forces an attacker to defeat W^X on cftype + kCFI on subsys ops + CAP_SYS_RESOURCE strict-userns + refcount saturation + integer-overflow trap before reaching a host-PID-exhaustion primitive.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- cgroup core (covered in `cgroup-core.md` Tier-3)
- cpuset controller (covered in `cpuset.md` Tier-3)
- memcg / memory controller (covered in `kernel/cgroup/memcg.md` future Tier-3)
- io controller (covered in `kernel/cgroup/io.md` future Tier-3)
- cpu controller (covered in `kernel/cgroup/cpu.md` future Tier-3)
- Implementation code
