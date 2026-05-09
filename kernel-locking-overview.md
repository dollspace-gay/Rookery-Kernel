---
title: "Tier-3: kernel/locking/00-overview — locking primitives hub"
tags: ["design-doc", "tier-3", "kernel"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 hub for the kernel's locking primitive family: `mutex`, `spinlock` (qspinlock + paravirt-aware), `rwsem`, `seqlock`, RCU (Tree RCU + SRCU + Tasks RCU + tiny variants), `qrwlock` (queued rw spinlock), `osq_lock` (optimistic spinning), `percpu-rwsem`, `rtmutex` (PI mutex), `refcount_t` (saturating), `atomic_t`/`atomic64_t`, and `lockdep` (the runtime lock-ordering validator).

Spawns sub-tier docs per `kernel/00-overview.md`: `mutex.md`, `spinlock.md`, `rwsem.md`, `seqlock.md`, `rcu.md`, `lockdep.md`. This 00-overview.md establishes the framework; sub-docs detail per-primitive implementation.

These primitives are referenced by **every** subsystem; the locking framework is the single most-cited piece of infrastructure in Rookery's design corpus.

### Requirements

- REQ-1: Mutex semantics: sleepable; FIFO acquisition order under contention via the optimistic-spin / wait-queue split; identical wakeup behavior. Cross-ref `mutex.md` Tier-3 sub-doc for detail.
- REQ-2: Spinlock semantics: queued spinlock (qspinlock) is the default; raw_spinlock_t is the legacy unqueued variant; paravirt-aware qspinlock when CONFIG_PARAVIRT_SPINLOCKS=y. PREEMPT_RT swaps spinlock_t for sleepable variant.
- REQ-3: rwsem semantics: writer-priority by default; readers and writers wait in FIFO when contended.
- REQ-4: seqlock semantics: writer increments seq (now odd), writes, increments seq (now even); reader retries on `seq & 1`.
- REQ-5: RCU: classic Tree RCU is the default; SRCU for sleepable read-side; Tasks RCU + Tasks Trace RCU for additional grace-period flavors. Each maintains the upstream-defined grace-period semantics.
- REQ-6: refcount_t saturating arithmetic per upstream; underflow/overflow attempts trigger WARN_ON.
- REQ-7: atomic_t / atomic64_t per upstream's API set + memory-ordering guarantees (`Ordering::Acquire` / `Release` / `SeqCst` / `Relaxed` mapped from upstream's `*_relaxed` / `*_acquire` / `*_release` suffix family).
- REQ-8: lockdep runtime validator: when CONFIG_LOCKDEP=y, every lock acquisition tracked; circular-dependency / recursive-locking / soft-lockup-detection reports format-identical to upstream.
- REQ-9: percpu-rwsem: optimized for the writer-rare case; readers traverse a per-CPU fast path; writer requires synchronize on every CPU's per-CPU state.
- REQ-10: rtmutex: full priority-inheritance + protocol; user-visible via PI-futex (cross-ref `kernel/futex.md`).
- REQ-11: Mandatory L2 TLA+ models per `kernel/00-overview.md` Layer 2: qspinlock, qrwlock, RCU grace-period detection, percpu_rwsem, futex_pi.
- REQ-12: Each primitive has Layer-1 SAFETY proofs for its `unsafe` blocks (atomic CAS, spin-poll, memory barriers).
- REQ-13: Cross-ref `kernel/sync` upstream rust-for-linux abstractions (`rust/kernel/sync/`); extend rather than replace.
- REQ-14: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: locktorture (`tools/testing/selftests/locking/locktorture.sh`) runs to completion with the same pass/fail set as upstream, exercising mutex, spinlock, rwsem, percpu-rwsem under stress. (covers REQ-1 through REQ-3, REQ-9, REQ-12)
- [ ] AC-2: rcutorture passes with the same set as upstream. (covers REQ-5, REQ-12)
- [ ] AC-3: A seqlock concurrent-reader-writer test produces no torn reads under 16-CPU stress. (covers REQ-4)
- [ ] AC-4: A refcount-overflow test (saturate from MAX) emits the upstream WARN_ON; a refcount-underflow test ditto. (covers REQ-6)
- [ ] AC-5: Atomic-ordering test exercises every `Acquire`/`Release`/`SeqCst`/`Relaxed` operation; ordering observed matches upstream. (covers REQ-7)
- [ ] AC-6: Boot CONFIG_LOCKDEP=y; trigger an ABBA inversion deliberately; dmesg report format-identical to upstream. (covers REQ-8)
- [ ] AC-7: PI-futex selftest (`tools/testing/selftests/futex/`) passes (cross-ref `kernel/futex.md`). (covers REQ-10)
- [ ] AC-8: `make tla` passes models for qspinlock, qrwlock, rcu_grace_period, percpu_rwsem, futex_pi. (covers REQ-11)
- [ ] AC-9: A grep over Rookery for `kernel::sync::*` shows reuse of upstream rust-for-linux abstractions; no parallel namespace. (covers REQ-13)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-14)

### Architecture

### Sub-tier doc layout

```
.design/kernel/locking/
  00-overview.md       ← this hub document
  mutex.md             ← Tier-3 sub-doc: mutex.c + osq_lock + ww_mutex
  spinlock.md          ← qspinlock + qrwlock + paravirt + raw_spinlock
  rwsem.md             ← rwsem + percpu-rwsem
  seqlock.md           ← seqcount + seqlock
  rcu.md               ← Tree RCU + SRCU + Tasks RCU + tiny variants
  lockdep.md           ← runtime ordering validator + lock-class machinery
  rtmutex.md           ← PI mutex + rwbase
  refcount.md          ← saturating atomic counter
  atomic.md            ← atomic_t / atomic64_t / memory ordering
```

(This Tier-3 hub establishes shared concepts; sub-docs flesh out per-primitive detail. They are sub-Tier-3 — same tier numerically, but organized in a hub-and-spoke pattern under this directory.)

### Rust module organization

Already largely in upstream rust-for-linux:
- `kernel::sync::Mutex<T>` (= `Lock<T, MutexBackend>`) — exists
- `kernel::sync::SpinLock<T>` (= `Lock<T, SpinLockBackend>`) — exists
- `kernel::sync::Arc<T>`, `kernel::sync::Atomic*`, `kernel::sync::Refcount` — exists
- `kernel::sync::CondVar`, `kernel::sync::Completion`, `kernel::sync::SetOnce<T>` — exists
- `kernel::sync::rcu::*` — exists (Guard + Token)
- `kernel::sync::Barrier::*` — exists

Rookery to author (per issue #4):
- `kernel::sync::RwSemaphore<T>` — sleepable rw lock
- `kernel::sync::PercpuRwSem<T>` — per-CPU rwsem
- `kernel::sync::SeqLock<T>` — seqlock wrapper
- `kernel::sync::RtMutex<T>` — PI mutex (cross-ref futex)

### Locking and concurrency

This component IS the concurrency framework. Each primitive implements an algorithm; this hub doesn't add concurrency on top.

### Error handling

- `Err(EINTR)` — sleepable mutex acquisition interrupted by signal
- `Err(ETIMEDOUT)` — timed-out acquisition (e.g., `mutex_lock_timeout`)
- `Err(EDEADLK)` — lockdep detected recursive locking attempt; only with CONFIG_LOCKDEP=y

### Out of Scope

- Per-primitive sub-docs are at the same tier; details there
- Userspace mutex / pthread_mutex (cross-ref `kernel/futex.md` for the kernel-side futex)
- 32-bit-only locking paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Mutex (sleepable mutex) | `kernel/locking/mutex.c`, `kernel/locking/mutex-debug.c`, `include/linux/mutex.h` |
| Queued spinlock (default fast path) + paravirt | `kernel/locking/qspinlock.c`, `kernel/locking/qspinlock_paravirt.h` |
| Queued rwlock | `kernel/locking/qrwlock.c` |
| MCS spinlock primitive (used internally by qspinlock + osq_lock) | `kernel/locking/mcs_spinlock.h` |
| Optimistic spinning queue | `kernel/locking/osq_lock.c` |
| Real-time mutex (PI) | `kernel/locking/rtmutex.c`, `kernel/locking/rtmutex_api.c`, `kernel/locking/rwbase_rt.c` |
| Per-CPU rwsem | `kernel/locking/percpu-rwsem.c` |
| Lockdep (runtime validator) | `kernel/locking/lockdep.c`, `kernel/locking/lockdep_proc.c` |
| Lock-events stats | `kernel/locking/lock_events.c` |
| Torture testing | `kernel/locking/locktorture.c` |
| Sleep-in-atomic detector | `kernel/locking/irqflag-debug.c` |
| RCU (Tree RCU, SRCU, Tasks RCU, tiny variants) | `kernel/rcu/tree.c`, `kernel/rcu/srcutree.c`, `kernel/rcu/sync.c`, `kernel/rcu/tasks.h`, `kernel/rcu/tiny.c` |
| Public API | `include/linux/{spinlock,mutex,rwsem,seqlock,rcupdate,atomic,refcount,lockdep,percpu-rwsem}.h` |

### compatibility contract

### Locking primitive ABI

Each primitive has a userspace-invisible ABI (kernel-internal), but their PERFORMANCE characteristics are observable via:
- `/proc/lockdep` content (when CONFIG_LOCKDEP=y)
- `/proc/lock_stat` content (when CONFIG_LOCK_STAT=y)
- Lockdep dmesg reports on lock-ordering inversions
- RCU-stall reports

These contents are format-identical to upstream so existing perf scripts + locktorture + RCU-torture pass identically.

### lockdep ordering reports

`include/linux/lockdep.h` defines lockdep's lock-class subkey machinery. Reports of "possible recursive locking detected", "circular locking dependency detected", etc. format-identical so out-of-tree CI scripts that grep dmesg work unchanged.

### RCU grace period

RCU's grace-period machinery exposes:
- `/proc/sys/kernel/rcu_*` sysctls (`rcu_normal`, `rcu_expedited`, `rcu_normal_after_boot`)
- `/sys/kernel/rcu/*` (when CONFIG_RCU_TRACE=y)
- RCU stall-warning format (when grace period stalls > timeout)

Format-identical.

### Refcount overflow detection

`include/linux/refcount.h` provides `refcount_t` — a saturating atomic counter that catches overflow + use-after-free. Saturating semantics MUST match upstream so out-of-tree code using `refcount_inc` doesn't observe semantic differences.

### verification

### Layer 1: Kani SAFETY proofs

Anticipated `unsafe` clusters across sub-docs:
- Atomic CAS spin-loops (qspinlock, MCS, osq_lock)
- Memory-barrier intrinsics in seqlock + RCU
- Per-CPU access in percpu-rwsem
- Wait-queue manipulation in mutex slow-path
- RCU callback list manipulation

### Layer 2: TLA+ models (mandatory per `kernel/00-overview.md` Layer 2)

- `models/kernel/locking/qspinlock.tla` — proves MCS-based queued spinlock fairness + safety. (Existing academic proof; Rookery ships the Rust-implementation-specific model.)
- `models/kernel/locking/qrwlock.tla` — queued rwlock fairness.
- `models/kernel/locking/rcu_grace_period.tla` — proves grace-period detection: a grace period ends only after every previously-running RCU read-side critical section has completed. THE most important locking-related model.
- `models/kernel/locking/percpu_rwsem.tla` — fast-path / slow-path correctness.
- `models/kernel/futex/futex_pi.tla` (cross-ref `kernel/futex.md`) — PI mutex priority-inheritance correctness.
- `models/kernel/locking/seqlock.tla` (NEW; co-owned with `lib/00-overview.md` § vdso-core.md) — proves writer-reader retry semantics for seqlock.
- `models/kernel/locking/rtmutex.tla` (NEW) — proves rtmutex priority-inheritance graph remains acyclic under all concurrent operations.

### Layer 3: Kani harnesses for data-structure invariants

Per the locking primitives:
- MCS spinlock node chain integrity
- mutex wait-queue ordering (FIFO under contention)
- rwsem reader/writer count consistency
- RCU callback list non-duplication
- lockdep lock-class graph (DAG; no cycles)

### Layer 4: Functional correctness (opt-in)

Strong opt-in candidates declared in `kernel/00-overview.md` Layer 4:
- **RCU grace-period detection algorithm** via Verus — high-leverage, well-known, formally interesting. The single most-impactful Layer-4 proof in the locking subsystem.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** (saturating arithmetic) | `kernel::sync::Refcount` is the canonical refcount type; `Atomic*::add` etc. on a `Refcount` saturates rather than wrapping | § Mandatory |

### Row-1 features consumed by this component

- **KERNEXEC**: locking-primitive code text is RX/RO
- **CONSTIFY**: lockdep lock-class strings, atomic-op tables are `static const`
- **SIZE_OVERFLOW**: refcount + sequence-counter arithmetic uses checked operators (saturating semantics IS the desired behavior here)
- **PRIVATE_KSTACKS**: per-CPU stacks isolated; per-CPU spinlocks operate on per-CPU MCS nodes

### Row-2 / GR-RBAC integration

Locking primitives don't invoke LSM hooks (they're below the LSM layer).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above; sub-docs flesh out per-primitive specifics.)

