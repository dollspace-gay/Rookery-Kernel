---
title: "Tier-3: kernel/locking/qspinlock.c — qspinlock (4-queue MCS slow-path + per-CPU pending-bit + paravirt extensions)"
tags: ["tier-3", "kernel-locking", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`qspinlock` is the kernel's universal spinlock implementation since 4.2 — replaces the legacy ticket-spinlock for every `spin_lock` call site (`raw_spinlock_t` and the bigger `spinlock_t` wrapper). 4-byte struct that fits in a single cacheline word; combines a fast-path single-cmpxchg uncontended acquire with an MCS-style queued slow-path on contention. Per-NUMA-node MCS queues organized as a 4-tail bit-encoding inside the lock word (no extra per-lock memory). Paravirt extension (CONFIG_PARAVIRT_SPINLOCKS) adds vCPU yield + IPI-kick mechanism so virtual machines don't burn CPU on spinning.

This Tier-3 covers `kernel/locking/qspinlock.c` (~400 lines) + paravirt header + MCS helper.

### Acceptance Criteria

- [ ] AC-1: Microbenchmark (single-CPU acquire/release) within 5% of upstream baseline cycle count.
- [ ] AC-2: Contention test: 64-CPU sustained acquire/release on shared lock; throughput within 5% of upstream baseline; no starvation observed.
- [ ] AC-3: Nested IRQ test: spinlock held in task context, IRQ fires + grabs different spinlock, NMI fires + grabs another — no MCS-node corruption (KASAN clean).
- [ ] AC-4: Paravirt test: KVM guest on overcommitted host (16 vCPUs on 8 pCPUs) with sustained lock contention shows pv_wait/pv_kick fires from `lock_events` tracepoints; throughput close to non-overcommitted scenario.
- [ ] AC-5: locktorture stress (100s sustained) with N=64 threads passes; no missed wakeup, no deadlock, no use-after-free.
- [ ] AC-6: Lockdep stress: deadlock detection between two qspinlocks correctly reported.
- [ ] AC-7: KUnit `qspinlock` tests pass.

### Architecture

`QSpinlock` lives in `kernel::sync::QSpinlock`:

```
#[repr(C)]
struct QSpinlock {
  val: AtomicU32,  // bits 0-7: locked-byte; bit 8-15: pending-byte; bits 16-31: tail (cpu+1, idx)
}
```

Layout of `val`:
- Bits 0..7: `_Q_LOCKED_VAL` (1 if locked, 0 if free)
- Bits 8..15: `_Q_PENDING_VAL` (1 if exactly one waiter is pending, 0 otherwise)
- Bits 16..31: `_Q_TAIL_*` (encoded `(cpu+1, idx)`; 0 = no queue tail)

Per-CPU MCS-node storage:

```
struct McsNode {
  next: AtomicPtr<McsNode>,
  locked: AtomicU32,
  count: u32,  // re-entry counter for nested-IRQ-context allocations
  // paravirt extension:
  cpu: AtomicI32,
  state: AtomicU8,  // pv state: vCPU running / hashed / halted
}

#[per_cpu]
static QNODES: [McsNode; MAX_NODES /* = 4 */];
```

Fast-path `QSpinlock::lock`:
```
loop {
  let mut val = self.val.load(Relaxed);
  if val == 0 {
    if self.val.compare_exchange_weak(0, _Q_LOCKED_VAL, Acquire, Relaxed).is_ok() {
      return;
    }
  } else {
    return self.slowpath(val);
  }
}
```

Slow-path `QSpinlock::slowpath(val)`:
1. If `val == _Q_LOCKED_VAL` (only locked, no pending, no queue):
   - CAS to set pending: `cmpxchg(val, _Q_LOCKED_VAL | _Q_PENDING_VAL)`.
   - On success: spin until `val.locked == 0`; CAS to take lock + clear pending. Return.
2. Else (some queue OR pending set):
   - Get our per-cpu MCS node: `node = QNODES[smp_processor_id()].at(context_idx)` where `context_idx ∈ {task=0, softirq=1, hardirq=2, nmi=3}` (incremented for nested-context).
   - `node.locked = 0; node.next = NULL`.
   - Encode our tail: `tail = encode_tail(cpu, context_idx)`.
   - XCHG tail-bits in lock word with our tail (atomic): `prev_tail = xchg(&val.tail, tail)`.
   - If `prev_tail != 0`: there's a predecessor; link `prev = decode_tail(prev_tail)`; `prev.next = node`; spin on `node.locked == 0`.
   - We're at queue head now. Spin until `val.locked == 0 && val.pending == 0`.
   - CAS lock-byte from 0 to 1, clearing tail if we're sole queue member.
   - Hand off to next: `next = node.next`; if `next != NULL`, set `next.locked = 1` (atomic store-release).
3. Return.

Unlock `QSpinlock::unlock`:
- `smp_store_release(&val.locked_byte, 0)` — single byte store; immediately released.

Paravirt `QSpinlock::pv_slowpath`:
- After ~16 spin iterations on `node.locked == 0`, call `pv_wait(&node.locked, 0)` → vCPU yields to host.
- On lock-handoff (`next.locked = 1`), if `next.cpu` was halted via pv_wait: `pv_kick(next.cpu)` IPI.
- Defense against host-preempted vCPU lock-holder: pv_wait gives the pCPU back to host; pv_kick wakes us when we should run again.

Per-CPU MCS node count tracks nested IRQ contexts: NMI can grab a spinlock that hardirq holds that softirq holds that task holds — each context gets its own MCS node. Bug-check: count > MAX_NODES = WARN + spin-without-queueing fallback.

### Out of Scope

- Per-architecture `xchg`/`cmpxchg` primitives (covered in `arch/x86/atomic.md` future Tier-3)
- `mutex` (covered in `kernel/locking/mutex.md` future Tier-3)
- `rwsem` (covered in `kernel/locking/rwsem.md` future Tier-3)
- `qrwlock` (covered in `kernel/locking/qrwlock.md` future Tier-3)
- `seqlock` (covered in `kernel/locking/seqlock.md` future Tier-3)
- `lockdep` (covered in `kernel/locking/lockdep.md` future Tier-3)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct qspinlock` (`u32 val`) | the entire lock state | `kernel::sync::QSpinlock` |
| `queued_spin_lock(&lock)` | acquire (fast-path inline + slow-path call) | `QSpinlock::lock` |
| `queued_spin_trylock(&lock)` | try-acquire | `QSpinlock::try_lock` |
| `queued_spin_unlock(&lock)` | release (single-byte store releasing locked-bit) | `QSpinlock::unlock` (Drop) |
| `queued_spin_lock_slowpath(&lock, val)` | the contention slow-path — full MCS queue dance | `QSpinlock::slowpath` |
| `qnodes` (per-cpu MCS node array) | per-cpu, per-context (4 contexts: task/softirq/hardirq/nmi) MCS queue node storage | `kernel::sync::PerCpuMcsNodes` |
| `__pv_queued_spin_lock_slowpath(&lock, val)` | paravirt slow-path with vCPU yield/wait | `QSpinlock::pv_slowpath` |
| `pv_wait(ptr, val)` | yield vCPU to host, expect notify | `arch::pv::wait` |
| `pv_kick(cpu)` | IPI-kick a yielded vCPU | `arch::pv::kick` |
| `is_pv_node` (paravirt helper) | per-MCS-node bit indicating waited via pv_wait | `McsNode::is_pv` |

### compatibility contract

REQ-1: `struct qspinlock` is exactly 4 bytes (u32) — preserves layout for every existing spinlock embedded in struct.

REQ-2: Fast-path: uncontended `cmpxchg` of `val` from 0 to 1 (locked-bit set, no pending, no tail) succeeds → return; ~1-2 cycles on cache-hit.

REQ-3: Pending-path: locked but no queue → set pending bit (`val.pending = 1`) → wait for locked-bit clear → CAS to take lock + clear pending. Avoids enqueueing single waiter.

REQ-4: Queued-path: locked + pending OR locked + tail-set → enqueue this CPU's MCS node at tail; spin on local cacheline (`node.locked = 0`); when prior tail's `node.next->locked = 1` set OR per-CPU MCS chain unblocks, transition to head. Head spins on lock-byte clear, takes lock + advances tail.

REQ-5: Per-cpu MCS node array sized 4 (per-context: task/softirq/hardirq/nmi) — supports lock_lock_lock_lock nested across IRQ contexts without node-reuse corruption.

REQ-6: Tail encoding in lock word: high 16 bits encode `(cpu+1, idx)`; allows 64K CPUs × 4 contexts = 256K simultaneous queue heads (more than any real system).

REQ-7: Paravirt extension: when CONFIG_PARAVIRT_SPINLOCKS=y AND running under hypervisor, slow-path waiters call `pv_wait(ptr, val)` after spinning N (16) times; lock-releaser kicks via `pv_kick(cpu)` (IPI). Defense against vCPU spinning while host is preempted.

REQ-8: Memory ordering: lock acquire = ACQUIRE (smp_load_acquire on lock-byte read in slow-path; cmpxchg-acquire in fast-path). Unlock = RELEASE (smp_store_release on lock-byte clear). Per-MCS-chain `node.locked` updates use ACQUIRE/RELEASE pairs.

REQ-9: Lockdep + lock-events instrumentation hooks preserved (per-acquire / per-release tracepoints).

REQ-10: `queued_spin_is_locked` + `queued_spin_value_unlocked` + `queued_spin_is_contended` introspection match upstream semantics.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mcs_node_no_oob` | OOB | per-cpu QNODES indexed by context_idx ∈ [0, 3]; never overflow. |
| `tail_encoding_no_overflow` | OVERFLOW | `encode_tail(cpu, idx)` checked against u16 bound; cpu < 64K, idx < 4. |
| `lock_word_atomic` | ATOMICITY | per-byte store of locked-byte (smp_store_release) preserves whole-word read-consistency for spin observers. |
| `cmpxchg_progress` | PROGRESS | fast-path cmpxchg-weak retried only on conflict, not spurious; eventual success guaranteed under non-contended scenario. |

### Layer 2: TLA+

`models/locking/qspinlock_mcs.tla` (declared in parent `kernel/locking/00-overview.md`): proves MCS queue invariants — every waiter eventually becomes head; head correctly hands off via `next.locked = 1`; no two waiters simultaneously enter critical section.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Lock-byte cleared by exactly one unlock per acquire (no double-unlock) | `QSpinlock::unlock` |
| Per-CPU MCS node `count` invariant: `count <= MAX_NODES` always; `count` decremented on slow-path return matched by increment on entry | `slowpath` enter/exit |
| `node.next` linked-list invariant: when waiter A enqueues after B, `B.next = A` after A's xchg returns prev=B; A's transition to head implies A.next = NULL (B already left) OR A was sole tail (xchg returned 0) | MCS link |

### Layer 4: Verus/Creusot functional

Mutual-exclusion: at any instant, at most one task / IRQ-context is in the critical section guarded by a given QSpinlock. Encoded as a Verus invariant on the lock-state plus per-context held-list: `forall lk. count(holders(lk)) <= 1`.

Liveness: every waiter eventually acquires (no starvation), bounded by N waiters ahead in queue.

### hardening

(Inherits row-1 features from `kernel/locking/00-overview.md` § Hardening.)

qspinlock-specific reinforcement:

- **Per-CPU MCS node count overflow check** — WARN_ONCE if count exceeds MAX_NODES (defense against unbounded nested-context recursion via locking bug).
- **Pending-bit + queue-tail mutual-exclusion** — only one of {pending, queue} can be set; CAS sequence ensures no race exposes "pending + queue tail simultaneously" state.
- **Memory ordering pairings** — every ACQUIRE matched with a corresponding RELEASE on the same byte (verifier ensures via Layer-2 TLA+).
- **Per-architecture cmpxchg correctness** — qspinlock relies on arch's `xchg`/`cmpxchg` being correctly seq-cst-ish (per-arch primitive validated via `arch/x86/atomic.md` future Tier-3).
- **Lockdep + lock-events tracepoint hooks** preserved — defense against lock ordering bugs via lockdep deadlock detector.
- **Paravirt vCPU yield bounded** — pv_wait timeout ensures even if pv_kick is missed (host bug), waiter eventually retries spinning + makes progress.
- **No allocator dependency** — all state in per-cpu static + lock word; no slab alloc → safe from inside slab/page allocator.

