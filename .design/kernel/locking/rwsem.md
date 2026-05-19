# Tier-3: kernel/locking/rwsem.c — rwsem (read-write semaphore)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/locking/00-overview.md
upstream-paths:
  - kernel/locking/rwsem.c (~1780 lines)
  - include/linux/rwsem.h
-->

## Summary

rwsem (read-write semaphore) is Linux's sleeping reader-writer lock: many readers can hold concurrently, but writer is exclusive. Per-`rw_semaphore` uses a single `atomic_long_t count` encoding {writer-bit, owner-bit, waiter-bit, count-of-readers}. Per-read-acquire fast-path: atomic-cmpxchg-add. Per-write-acquire fast-path: atomic-cmpxchg to writer-state. Per-slow-path: queue on wait_list (FIFO + writer-prio anti-starve), sleep. Per-optimistic-spin: writer-spin if owner is on-CPU (MCS lock + osq_lock). Per-reader-bias: per-CPU "reader-fast-path" bias counter for highly-contended read paths. Critical for: per-fs i_rwsem, per-mm mmap_lock, per-namespace sem_locks.

This Tier-3 covers `rwsem.c` (~1780 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rw_semaphore` | per-rwsem state | `RwSemaphore` |
| `__init_rwsem()` | per-rwsem init | `Rwsem::init` |
| `down_read()` / `down_read_killable()` | per-reader acquire (blocking) | `Rwsem::down_read` |
| `down_write()` / `down_write_killable()` | per-writer acquire (blocking) | `Rwsem::down_write` |
| `down_read_trylock()` | per-reader try-acquire | `Rwsem::down_read_trylock` |
| `down_write_trylock()` | per-writer try-acquire | `Rwsem::down_write_trylock` |
| `up_read()` | per-reader release | `Rwsem::up_read` |
| `up_write()` | per-writer release | `Rwsem::up_write` |
| `downgrade_write()` | per-writer→reader | `Rwsem::downgrade_write` |
| `__down_read()` / `__up_read()` | per-fast-path helpers | `Rwsem::__down_read` / `__up_read` |
| `rwsem_down_read_slowpath()` | per-slow-path | `Rwsem::down_read_slowpath` |
| `rwsem_down_write_slowpath()` | per-slow-path | `Rwsem::down_write_slowpath` |
| `rwsem_optimistic_spin()` | per-OSQ spin | `Rwsem::optimistic_spin` |
| `rwsem_owner_is_writer()` | per-owner-ptr check | `Rwsem::owner_is_writer` |
| `RWSEM_WRITER_LOCKED` / `RWSEM_WRITER_OWNED` / `RWSEM_FLAG_WAITERS` / `RWSEM_FLAG_HANDOFF` / `RWSEM_READER_BIAS` / `RWSEM_READER_OWNED` | count-bits | shared |

## Compatibility contract

REQ-1: rw_semaphore layout:
- count: atomic_long_t (multi-state encoding).
- owner: atomic_long_t (writer task-pointer when locked).
- osq: OptimisticSpinQueue (MCS-like for spin).
- wait_list: ListHead of waiters.
- wait_lock: raw_spinlock_t.

REQ-2: Per-count encoding (64-bit):
- Bit[0]: RWSEM_WRITER_LOCKED.
- Bit[1]: RWSEM_FLAG_WAITERS (sleepers).
- Bit[2]: RWSEM_FLAG_HANDOFF (handoff to writer pending).
- Bits[3:7]: reserved.
- Bits[8:63]: reader count (RWSEM_READER_BIAS unit).

REQ-3: Per-owner encoding:
- LSB=0: writer-owned, ptr to task_struct.
- LSB=1 (RWSEM_READER_OWNED): reader-owned (no specific owner).
- LSB=2 (RWSEM_NONSPINNABLE): readers should not spin-wait.

REQ-4: down_read fast-path:
- count_old = atomic_long_fetch_add_acquire(RWSEM_READER_BIAS, &sem.count).
- if !(count_old & RWSEM_WRITER_LOCKED ∨ FLAG_WAITERS): return.
- rwsem_down_read_slowpath (block).

REQ-5: down_write fast-path:
- if atomic_long_try_cmpxchg(&sem.count, 0, RWSEM_WRITER_LOCKED): set owner; return.
- rwsem_down_write_slowpath.

REQ-6: Per-slowpath:
- Per-writer: enter osq optimistic spin if owner running.
- if owner not running ∨ spin-timeout: queue on wait_list; schedule.

REQ-7: up_read:
- count_new = atomic_long_sub_return_release(RWSEM_READER_BIAS, &sem.count).
- if (count_new & FLAG_WAITERS) ∧ reader-count == 0: wake writer.

REQ-8: up_write:
- atomic_long_andnot(RWSEM_WRITER_LOCKED, &sem.count).
- atomic_long_set(&sem.owner, 0).
- if FLAG_WAITERS: rwsem_wake.

REQ-9: downgrade_write:
- count = atomic_long_sub_return_release(RWSEM_WRITER_LOCKED - RWSEM_READER_BIAS, &sem.count).
- owner_set_reader_bias.
- if FLAG_WAITERS: wake readers.

REQ-10: Per-handoff (anti-starvation):
- After per-writer waits ≥ RWSEM_WAIT_TIMEOUT: set FLAG_HANDOFF.
- New readers stop reading; let next-waiter (writer) acquire.

REQ-11: Per-OSQ:
- osq_lock_init.
- Per-writer slow-path: osq_lock; spin while owner running.
- If owner sleeps or owner-changes: osq_unlock; fall to sleep-queue.

## Acceptance Criteria

- [ ] AC-1: __init_rwsem: count = 0; wait_list empty.
- [ ] AC-2: Single-reader down_read + up_read: count returns to 0.
- [ ] AC-3: Multi-reader concurrent: count grows; no writer-bit.
- [ ] AC-4: down_write while readers held: queues; sleeps until reader-count zero.
- [ ] AC-5: down_read while writer held: queues; sleeps.
- [ ] AC-6: downgrade_write: writer-bit cleared; reader-bias incremented; queued readers woken.
- [ ] AC-7: down_read_trylock with writer held: returns 0 (no block).
- [ ] AC-8: down_write_trylock with reader held: returns 0.
- [ ] AC-9: Per-handoff: writer-waited long; new readers see HANDOFF + queue.
- [ ] AC-10: down_read_killable interrupted by SIGKILL: returns -EINTR.
- [ ] AC-11: rwsem_optimistic_spin: per-writer-spin while owner-running.

## Architecture

Per-rwsem:

```
struct RwSemaphore {
  count: AtomicLong,
  owner: AtomicLong,
  #[cfg(CONFIG_RWSEM_SPIN_ON_OWNER)]
  osq: OptimisticSpinQueue,
  wait_lock: RawSpinLock,
  wait_list: ListHead<RwsemWaiter>,
}

struct RwsemWaiter {
  list: ListLink,
  task: *TaskStruct,
  type_: RwsemWaiterType,                        // READER / WRITER
  timeout: u64,
  last_rowner: u64,
}

const RWSEM_WRITER_LOCKED: u64 = 1 << 0;
const RWSEM_FLAG_WAITERS: u64 = 1 << 1;
const RWSEM_FLAG_HANDOFF: u64 = 1 << 2;
const RWSEM_READER_SHIFT: u32 = 8;
const RWSEM_READER_BIAS: u64 = 1 << RWSEM_READER_SHIFT;
const RWSEM_READER_MASK: u64 = !RWSEM_READER_BIAS_MASK;

const RWSEM_READER_OWNED: u64 = 1 << 0;          // owner LSB
const RWSEM_NONSPINNABLE: u64 = 1 << 1;
```

`Rwsem::down_read(sem)`:
1. count_old = atomic_long_fetch_add_acquire(RWSEM_READER_BIAS, &sem.count).
2. if count_old & (RWSEM_WRITER_LOCKED | RWSEM_FLAG_HANDOFF):
   - Rwsem::down_read_slowpath(sem, count_old + RWSEM_READER_BIAS, TASK_UNINTERRUPTIBLE).
3. /* Read access granted */
4. atomic_long_or(RWSEM_READER_OWNED, &sem.owner).

`Rwsem::down_write(sem)`:
1. if !atomic_long_try_cmpxchg_acquire(&sem.count, 0, RWSEM_WRITER_LOCKED):
   - Rwsem::down_write_slowpath(sem, TASK_UNINTERRUPTIBLE).
2. atomic_long_set(&sem.owner, current as u64).

`Rwsem::down_read_slowpath(sem, count, state) -> Result<()>`:
1. waiter.task = current; waiter.type = READER.
2. raw_spin_lock_irq(&sem.wait_lock).
3. /* Insert into wait_list */
4. list_add_tail(&waiter.list, &sem.wait_list).
5. /* If !FLAG_WAITERS, set */
6. atomic_long_or(RWSEM_FLAG_WAITERS, &sem.count).
7. raw_spin_unlock_irq(&sem.wait_lock).
8. /* Sleep */
9. for { set_current_state(state); if rwsem_can_continue(sem, READER): break; schedule(); }
10. list_del(&waiter.list).
11. Done.

`Rwsem::down_write_slowpath(sem, state) -> Result<()>`:
1. waiter.task = current; waiter.type = WRITER.
2. /* Optimistic spin */
3. if rwsem_optimistic_spin(sem): return Ok.
4. /* Queue + sleep */
5. raw_spin_lock_irq(&sem.wait_lock).
6. list_add_tail(&waiter.list, &sem.wait_list).
7. atomic_long_or(RWSEM_FLAG_WAITERS, &sem.count).
8. raw_spin_unlock.
9. /* Wait for HANDOFF */
10. for { set_current_state(state); if rwsem_try_acquire_writer(sem): break; schedule(); }
11. list_del.

`Rwsem::up_read(sem)`:
1. count_new = atomic_long_sub_return_release(RWSEM_READER_BIAS, &sem.count).
2. if (count_new & RWSEM_FLAG_WAITERS) ∧ (reader_count(count_new) == 0):
   - rwsem_wake(sem).

`Rwsem::up_write(sem)`:
1. atomic_long_andnot(RWSEM_WRITER_LOCKED, &sem.count).
2. atomic_long_set(&sem.owner, 0).
3. if count & RWSEM_FLAG_WAITERS: rwsem_wake(sem).

`Rwsem::downgrade_write(sem)`:
1. count = atomic_long_sub_return_release(RWSEM_WRITER_LOCKED - RWSEM_READER_BIAS, &sem.count).
2. atomic_long_set(&sem.owner, current as u64 | RWSEM_READER_OWNED).
3. if count & RWSEM_FLAG_WAITERS: rwsem_wake_readers(sem).

`Rwsem::optimistic_spin(sem) -> bool`:
1. osq_lock(&sem.osq).
2. while owner-on-CPU(&sem.owner):
   - if atomic_long_try_cmpxchg(&sem.count, 0, RWSEM_WRITER_LOCKED): goto acquired.
   - cpu_relax().
3. osq_unlock(&sem.osq).
4. return false.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `writer_excl_reader` | INVARIANT | (count & WRITER_LOCKED) ⟹ (count >> READER_SHIFT) == 0. |
| `owner_set_iff_writer_held` | INVARIANT | (count & WRITER_LOCKED) ⟺ (sem.owner != 0 ∧ LSB == 0). |
| `wait_list_consistent` | INVARIANT | (count & FLAG_WAITERS) ⟺ wait_list non-empty. |
| `count_reader_count_increments_on_down_read` | INVARIANT | per-down_read: count += READER_BIAS. |
| `handoff_implies_writer_waiting` | INVARIANT | (count & FLAG_HANDOFF) ⟹ writer at head of wait_list. |

### Layer 2: TLA+

`kernel/locking/rwsem.tla`:
- Per-down_read/write + per-up + per-slow-path + per-handoff.
- Properties:
  - `safety_writer_exclusive` — at most one writer held at any time.
  - `safety_readers_concurrent` — multiple readers OK.
  - `safety_no_starvation` — per-handoff bounded wait for writer.
  - `liveness_release_eventually_wakes_waiter` — per-up_write: waiter eventually wakes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Rwsem::down_read` post: count.reader_count++; owner.READER_OWNED bit set | `Rwsem::down_read` |
| `Rwsem::down_write` post: count.WRITER_LOCKED set; owner = current | `Rwsem::down_write` |
| `Rwsem::up_read` post: count.reader_count--; FLAG_WAITERS handled | `Rwsem::up_read` |
| `Rwsem::up_write` post: count.WRITER_LOCKED cleared; owner = 0; waiters woken | `Rwsem::up_write` |
| `Rwsem::downgrade_write` post: writer→reader; waiters-readers woken | `Rwsem::downgrade_write` |

### Layer 4: Verus/Creusot functional

`Per-rwsem: many readers OR one writer, mutually-exclusive; per-handoff prevents writer starvation; per-OSQ optimistic-spin for writer fast-path` semantic equivalence: per-Linux rwsem semantics (Documentation/locking/rwsem.rst).

## Hardening

(Inherits row-1 features from `kernel/locking/00-overview.md` § Hardening.)

rwsem-specific reinforcement:

- **Per-count atomic** — defense against per-state torn-update.
- **Per-OSQ optimistic spin gated by CONFIG_RWSEM_SPIN_ON_OWNER** — defense against per-CPU spin storm.
- **Per-HANDOFF flag prevents writer-starve** — defense against per-reader-flood writer-block.
- **Per-wait_list FIFO** — defense against per-priority-inversion.
- **Per-PREEMPT_RT alt: rt_mutex-backed rwsem** — defense against per-non-RT priority-inversion.
- **Per-owner ptr atomic** — defense against per-debug-mismatch with sem.count.
- **Per-task spin-on-owner cap** — defense against per-task on-CPU runaway-spin.
- **Per-killable / interruptible variants** — defense against per-SIGKILL hang.
- **Per-wake limits** — defense against per-mass-wake stampede.
- **Per-down_read_trylock atomic** — defense against per-CPU race.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on task/cred/sched_attr buffers carried under mmap_lock (the largest rwsem consumer).
- **PAX_KERNEXEC** — W^X for BPF JIT'd code, kprobe/uprobe trampolines invoked under rwsem read-side.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization; obscures `RwsemWaiter`-on-stack layout.
- **PAX_REFCOUNT** — saturating refcount on task_struct (the writer-owner ptr), files_struct, cred, mm_struct.
- **PAX_MEMORY_SANITIZE** — zero-on-free for task_struct; prevents reader-owner stale-pointer aliasing across slab reuse.
- **PAX_MEMORY_STACKLEAK** — kernel-stack zeroing on syscall exit clears leaked `RwsemWaiter` stack frames after slow-path return.
- **PAX_UDEREF** — strict user-pointer access for all task-pointer ops, including mmap_lock-held copy_from_user paths.
- **PAX_RAP / kCFI** — indirect-call signature enforcement for OSQ + wake-callback vtables.
- **GRKERNSEC_HIDESYM** — hide kernel addresses in /proc/<pid>/* + kallsyms; suppresses writer-owner ptr disclosure via /proc/lock_stat.
- **GRKERNSEC_HARDEN_PTRACE** — restrict ptrace cross-uid (Yama scope ≥ 1); blocks reading mmap_lock owner during a victim's page-fault.
- **GRKERNSEC_BRUTE** — exponential delay on consecutive brute attempts; throttles reader-flood writer-starvation probes.
- **GRKERNSEC_KSTACKOVERFLOW** — kernel-stack overflow guard against deep rwsem-held recursion (e.g. nested mmap_lock under fault path).
- **GRKERNSEC_DMESG** — restrict syslog so rwsem deadlock WARN traces leaking owner pointers are unreadable to unprivileged users.
- **GRKERNSEC_SYSCTL_DISABLE** — disable dangerous sysctls (e.g. /proc/sys/kernel/lock_stat) by default.
- **GRKERNSEC_CONFIG_AUDIT** — boot-time runtime-config integrity check; verifies RWSEM_SPIN_ON_OWNER setting matches signed profile.

Per-doc rationale: rwsem is the backbone of mmap_lock and i_rwsem, which gate page-fault and filesystem paths; a stale writer-owner pointer or reader-count corruption directly translates into UAF in the address-space or inode layer. PaX MEMORY_SANITIZE + REFCOUNT close the slab-reuse aliasing, RAP/kCFI hardens the OSQ + wake-callback dispatch, and Grsecurity HIDESYM/DMESG suppresses the owner-ptr leak surface that lock_stat would otherwise expose to attackers fingerprinting writer identity.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/locking/{spinlock, qspinlock, mutex}.c (covered in respective Tier-3)
- kernel/locking/osq_lock.c (covered separately if expanded)
- rt-mutex (covered separately)
- lockdep (covered separately)
- Implementation code
