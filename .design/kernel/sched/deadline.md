# Tier-3: kernel/sched/deadline — SCHED_DEADLINE (EDF + CBS)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - kernel/sched/deadline.c
  - kernel/sched/cpudeadline.c
  - kernel/sched/cpudeadline.h
  - include/linux/sched/deadline.h
  - include/uapi/linux/sched.h
  - include/uapi/linux/sched/types.h
-->

## Summary
Tier-3 design for `SCHED_DEADLINE` — Earliest Deadline First (EDF) scheduling with the Constant Bandwidth Server (CBS) wrapper for temporal isolation. The highest-priority normal scheduling class on Linux: outranks `SCHED_FIFO`/`SCHED_RR`/`SCHED_OTHER`, only `stop_task` outranks it.

Each SCHED_DEADLINE task declares a triple `(runtime, deadline, period)`: it requests `runtime` time units within every `period`, with each instance's relative deadline `deadline` from job-release. The kernel admits the task only if the system has enough total bandwidth (Σ runtime/period ≤ admissible-fraction); admission test is the kernel's bandwidth gate.

Sub-tier-3 of `kernel/sched/00-overview.md`. Sibling of `kernel/sched/rt.md` and `kernel/sched/cfs.md`. Used by audio/video pipelines (PipeWire), ROS 2 latency-bounded nodes, industrial real-time, and academic real-time research.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| SCHED_DEADLINE class core: EDF-tree, CBS wrapper, replenish/throttle, push/pull | `kernel/sched/deadline.c` |
| Per-CPU deadline tracking for push-decision | `kernel/sched/cpudeadline.c`, `kernel/sched/cpudeadline.h` |
| Public API | `include/linux/sched/deadline.h` |
| UAPI | `include/uapi/linux/sched.h` (SCHED_DEADLINE=6), `include/uapi/linux/sched/types.h` (sched_attr) |

## Compatibility contract

### `sched_setattr(pid, &sched_attr, flags)` — the only way to set SCHED_DEADLINE

`struct sched_attr` carries `sched_policy=SCHED_DEADLINE`, `sched_runtime`, `sched_deadline`, `sched_period`, plus `sched_flags` (SCHED_FLAG_DL_OVERRUN to opt into BANDWIDTH_OVERRUN signaling).

Constraints (identical to upstream + POSIX):

```
runtime > 0
deadline >= runtime
period >= deadline (or period == 0 ⇒ period := deadline)
runtime / period ≤ kernel admission threshold
```

Admission threshold defaults: `sched_rt_runtime_us / sched_rt_period_us` (95% by default) shared with RT class — total RT+DL bandwidth ≤ 95%.

### Per-cpu admission accounting

Adding a SCHED_DEADLINE task on CPU N consumes `runtime/period` of CPU N's admissible bandwidth. If pinned (cpuset) or migration-disallowed, the entire (runtime/period) charges that one CPU; if migration-allowed (the default), it charges the global root-domain's CPUs.

### `/proc/sys/kernel/sched_deadline_*`

- `sched_deadline_period_min_us` (default 100us): lower bound on `sched_period`
- `sched_deadline_period_max_us` (default 4s): upper bound

Identical defaults so userspace tooling (chrt, libdl examples) works unchanged.

### `struct sched_dl_entity` layout

`include/linux/sched.h`: per-task DL entity. Contains `rb_node` (DL runqueue rb-tree linkage), `dl_runtime`, `dl_deadline`, `dl_period`, `dl_bw`, `dl_density`, `runtime`, `deadline`, `flags`, `dl_throttled`, `dl_yielded`, replenish timer. Layout-equivalent.

### `struct dl_rq` layout

`kernel/sched/sched.h`: per-CPU DL state. Contains `root_rb` (rb-tree of runnable DL entities ordered by deadline), `running_bw`, `this_bw`, `total_bw`, `extra_bw`, `bw_ratio`. Layout-equivalent.

### push/pull: cpudeadline tracking

cpudeadline is a max-heap of (CPU, latest-deadline) pairs; `cpudl_find` returns CPU mask of CPUs running the latest-deadline tasks (worst victims for push). Algorithm + structure identical.

## Requirements

- REQ-1: `SCHED_DEADLINE` policy constant `=6`; `SCHED_FLAG_DL_OVERRUN` etc. identical bits.
- REQ-2: `sched_setattr` with policy=DEADLINE: validates triple constraints; runs admission test; admits-or-rejects with EBUSY identically.
- REQ-3: Admission test: per-cpu (or per-root-domain for migrate-allowed) bandwidth check `Σ runtime/period ≤ kernel cap`. Identical algorithm.
- REQ-4: EDF runqueue: rb-tree keyed on `dl_entity.deadline`; pick-next selects leftmost. Identical.
- REQ-5: Constant Bandwidth Server (CBS): per-task `runtime` decreases as it executes; on exhaustion → throttled; replenish-timer at next-`deadline` re-fills `runtime` and re-adds task. Identical algorithm.
- REQ-6: `SCHED_FLAG_DL_OVERRUN`: when task uses up `runtime` before deadline, generate SIGXCPU signal (or per-task callback). Identical.
- REQ-7: push/pull migration: `push_dl_task` migrates to cpudl-found CPU; `pull_dl_task` pulls from overload CPUs. Identical.
- REQ-8: Bandwidth donation in priority inheritance: when DL task blocks on rt_mutex held by lower-prio task, lender boosted to DL-tag (cross-ref `kernel/locking/rtmutex.md`).
- REQ-9: cpuset / cgroup interaction: setting `cpu.cfs_*` on a cgroup containing DL tasks is rejected; DL tasks bypass cfs bandwidth. Identical.
- REQ-10: `/proc/sys/kernel/sched_deadline_period_{min,max}_us` tunables work identically.
- REQ-11: `cpudeadline` max-heap maintained: O(log n) insert/decrease; `cpudl_find` O(1) given heap-top; identical.
- REQ-12: TLA+ model `models/sched/dl_class.tla` (mandatory per `kernel/sched/00-overview.md`) proves: with admission accepting only Σ runtime/period ≤ U_admission, every admitted task meets its deadlines under EDF.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `chrt --deadline --runtime 10000 --deadline 100000 --period 100000 -- /bin/some-rt-prog` admits and runs; `chrt -p $$` reports DEADLINE policy with the triple. (covers REQ-1, REQ-2)
- [ ] AC-2: An admission-impossible request (`runtime=900ms, period=1000ms` on a single-cpu pinned set with the 95% cap) returns EBUSY. (covers REQ-3)
- [ ] AC-3: Two SCHED_DEADLINE tasks with deadlines `100ms` and `200ms` from time T → leftmost-deadline (100ms) runs first; pick-next-task verifiable via tracepoint `sched:sched_switch`. (covers REQ-4)
- [ ] AC-4: A DL task hitting `runtime` exhaustion before its `deadline` → throttled until replenishment; observable via `cat /proc/<pid>/sched | grep dl`. (covers REQ-5)
- [ ] AC-5: With `SCHED_FLAG_DL_OVERRUN` set, runtime exhaustion delivers SIGXCPU. (covers REQ-6)
- [ ] AC-6: A 4-CPU machine, 4 DL tasks (each `runtime=10ms, period=20ms`), all spawned on CPU 0 → push-migration distributes them across all 4 CPUs; cpudl-heap maintains correct top. (covers REQ-7, REQ-11)
- [ ] AC-7: A DL task blocked on rt_mutex held by SCHED_OTHER task → lender boosted; cross-ref test in rt_mutex Tier-3. (covers REQ-8)
- [ ] AC-8: Attempting to attach a SCHED_DEADLINE task to a cgroup with non-default cfs bandwidth fails or cfs bandwidth on that cgroup is reset; documented behavior matches upstream. (covers REQ-9)
- [ ] AC-9: TLA+ `models/sched/dl_class.tla` proves: ∀ admitted task `t`, `t`'s execution-time within each period ≥ runtime AND each instance completes by deadline. (covers REQ-12)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::sched::deadline::DlClass` — main class entrypoint
- `kernel::sched::deadline::queue::DlRq` — per-CPU EDF rb-tree
- `kernel::sched::deadline::cbs::Cbs` — Constant Bandwidth Server
- `kernel::sched::deadline::admit::Admission` — admission test
- `kernel::sched::deadline::push_pull::DlPushPull` — migration
- `kernel::sched::deadline::cpudl::CpuDl` — per-CPU max-heap of latest deadlines
- `kernel::sched::deadline::overrun::OverrunSignaler` — SIGXCPU on overrun

### Locking and concurrency

- **Per-CPU rq lock**: protects dl_rq state
- **`cpudl->lock`** (cpudeadline max-heap): per-root-domain lock
- **`def_rt_bandwidth.rt_runtime_lock`** (shared with RT class for total cap)

push/pull uses `double_lock_balance` (same pattern as RT class).

### Error handling

- `Err(EINVAL)` — bad triple constraints (deadline < runtime, period < deadline, etc.)
- `Err(EBUSY)` — admission test failure
- `Err(EPERM)` — non-CAP_SYS_NICE attempting DL for non-self
- `Err(ESRCH)` — pid not found

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| EDF rb-tree insert/erase | `kani::proofs::sched::dl::rb_tree_safety` |
| CBS replenish-timer arithmetic (no underflow on `runtime -=`) | `kani::proofs::sched::dl::cbs_safety` |
| Admission-test bandwidth arithmetic (overflow-checked Σ) | `kani::proofs::sched::dl::admit_safety` |
| cpudl max-heap up/down operations | `kani::proofs::sched::dl::cpudl_safety` |
| push/pull double-lock | `kani::proofs::sched::dl::push_pull_safety` |

### Layer 2: TLA+ models

- `models/sched/dl_class.tla` (mandatory per `kernel/sched/00-overview.md` — DL admission-soundness proof) — proves: admission test correctness + per-task deadline-meeting under EDF. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| EDF rb-tree | leftmost node has minimum deadline; tree is bst-ordered + red-black-balanced | `kani::proofs::sched::dl::rb_invariants` |
| cpudeadline max-heap | parent's `latest_deadline` ≥ both children's | `kani::proofs::sched::dl::cpudl_invariants` |
| `dl_bw->total_bw` | equals Σ admitted-task `bw` (runtime/period) on each rq | `kani::proofs::sched::dl::bw_invariants` |

### Layer 4: Functional correctness (opt-in)

- **EDF schedulability theorem** via Verus — proves: ∀ task set `T` with Σ runtime/period ≤ U_admission and individual deadlines, EDF on `T` meets all deadlines (Liu & Layland 1973).
- **CBS isolation theorem** via Verus — proves: a misbehaving DL task that exhausts its runtime before deadline does not steal CPU from other DL tasks; throttling enforces budget.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | dl_bw + dl_entity refcounts use `Refcount` (saturating) | § Mandatory |
| **SIZE_OVERFLOW** | runtime / deadline / period 64-bit arithmetic uses checked operators (admission Σ, CBS replenish) | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-task dl_entity slabs
- **CONSTIFY**: `dl_sched_class` ops vtable static const
- **SIZE_OVERFLOW**: see above

### Row-2 / GR-RBAC integration

- LSM hook `security_task_setscheduler` (shared with RT path); GR-RBAC policy can deny SCHED_DEADLINE per-subject.
- Default useful policy: deny SCHED_DEADLINE outside gradm-marked roles, since admission consumes systemwide bandwidth and a malicious admit can deny CFS to others. Default policy: empty so behavior matches upstream until policy is loaded.
- DoS-prevention: admission test cap `sched_rt_runtime_us / sched_rt_period_us` (default 95%) prevents Σ DL+RT bandwidth from starving CFS.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `sched_attr` copies for `sched_setattr`/`getattr` bounds-checked at the user boundary; no `sched_dl_entity` slab leakage via short reads.
- **PAX_KERNEXEC** — `dl_sched_class` vtable resides in read-only W^X kernel text; `enqueue_task_dl` / `task_tick_dl` cannot be patched live.
- **PAX_RANDKSTACK** — per-syscall kstack offset so DL admission grooming via repeated `sched_setattr` cannot exploit predictable stack layout.
- **PAX_REFCOUNT** — `dl_bw` total-bandwidth counter and `dl_rq->dl_nr_running` use saturating refcount; admission overflow oopses rather than silently over-admits.
- **PAX_MEMORY_SANITIZE** — freed `task_struct` DL fields (`dl_se`) scrubbed so a reused task slot cannot inherit stale `dl_runtime`/`dl_deadline`.
- **PAX_UDEREF** — DL bandwidth control reads cgroup attributes via kernel mappings only; no implicit user-page reach during admission.
- **PAX_RAP / kCFI** — `sched_class` function-pointer set type-signatured; mismatched `pick_next_task_dl`/`task_fork_dl` signature is a hard CFI fault.
- **GRKERNSEC_HIDESYM** — `dl_bw`, `dl_rq`, and rb-tree internal addresses hidden from `/proc/kallsyms` and dmesg leaks.
- **GRKERNSEC_DMESG** — DL admission rejection / overrun WARN backtraces gated to CAP_SYSLOG.
- **SCHED_DEADLINE CAP_SYS_NICE** — switching a task into `SCHED_DEADLINE` and tightening `dl_runtime`/`dl_period` requires CAP_SYS_NICE so unprivileged users cannot starve `SCHED_OTHER`.
- **Admission control bounded** — total `dl_bw` per root domain capped at `sched_rt_runtime_us / sched_rt_period_us`-derived ceiling; PAX_REFCOUNT keeps the running sum from wrapping under hostile `sched_setattr` flooding.
- **DL_OVERRUN signal** — `SIGXCPU` (or DL_OVERRUN-flagged SIGXCPU) raised through the standard `send_signal` path so overrun handling cannot be redirected; signal_struct refcounted under PAX_REFCOUNT.
- **Rationale** — `SCHED_DEADLINE` directly controls CPU time and IRQ-context preemption; an attacker who can corrupt admission, the EDF rb-tree, or the bandwidth accumulator can starve the system or escalate into kernel-mode redirection. Grsec/PaX hardening enforces that even root must traverse capability, refcount, and CFI gates before influencing the deadline class.

## Open Questions

(none — SCHED_DEADLINE semantics are exhaustively specified by upstream + the underlying EDF/CBS literature)

## Out of Scope

- SCHED_FIFO / SCHED_RR (cross-ref `kernel/sched/rt.md`)
- SCHED_OTHER / SCHED_BATCH / SCHED_IDLE (cross-ref `kernel/sched/cfs.md`)
- 32-bit-only paths
- Implementation code
