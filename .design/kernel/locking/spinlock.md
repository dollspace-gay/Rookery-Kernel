# Tier-3: kernel/locking/spinlock.c — generic spinlock API (raw-spinlock + spin_lock dispatch + lockdep)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/locking/00-overview.md
upstream-paths:
  - kernel/locking/spinlock.c
  - kernel/locking/spinlock_debug.c
  - include/linux/spinlock.h
  - include/linux/spinlock_types.h
  - include/linux/spinlock_api_smp.h
  - include/linux/spinlock_api_up.h
-->

## Summary

`spinlock_t` is the kernel's most-used short-term mutual-exclusion primitive (~10000+ uses in tree). On x86_64 the arch backend is qspinlock (covered in `qspinlock.md` Tier-3); on PREEMPT_RT, `spinlock_t` is a sleepable rt_mutex while `raw_spinlock_t` retains true-spin semantics. spinlock.c is the arch-agnostic dispatcher: `spin_lock` / `spin_lock_irq` / `spin_lock_irqsave` / `spin_lock_bh` / `spin_trylock` thin wrappers that disable preempt + call arch-spinlock + integrate with lockdep + lockstat. Per-call-site lockdep tracking uses static `struct lock_class_key` to validate ordering at runtime.

This Tier-3 covers `spinlock.c` (~427 lines) + `spinlock.h` (~678 lines) + spinlock_types.h.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `spinlock_t` | core spinlock type | `kernel::locking::spinlock::SpinLock<T>` |
| `raw_spinlock_t` | non-preemptible variant | `RawSpinLock<T>` |
| `spin_lock_init(lock)` / `raw_spin_lock_init(lock)` | per-instance init | `SpinLock::new` / `RawSpinLock::new` |
| `spin_lock(lock)` | acquire (preempt-disable + arch-acquire) | `SpinLock::lock` |
| `spin_unlock(lock)` | release | `SpinLock::unlock` |
| `spin_lock_irq(lock)` | acquire + IRQ-disable | `SpinLock::lock_irq` |
| `spin_lock_irqsave(lock, flags)` | acquire + save+disable IRQ | `SpinLock::lock_irqsave` |
| `spin_unlock_irqrestore(lock, flags)` | release + restore IRQ | `SpinLock::unlock_irqrestore` |
| `spin_lock_bh(lock)` | acquire + BH-disable | `SpinLock::lock_bh` |
| `spin_trylock(lock)` | non-blocking acquire | `SpinLock::try_lock` |
| `spin_is_locked(lock)` | read-only state query | `SpinLock::is_locked` |
| `spin_can_lock(lock)` | spec query (lockdep) | `SpinLock::can_lock` |
| `read_lock(rwlock)` / `write_lock(rwlock)` | rwlock_t variants | `RwLock::read` / `_write` |
| `spinlock_check(lock)` | lockdep + LOCKDEP_RECURSION_LIMIT check | `SpinLock::check_lockdep` |
| `__raw_spin_lock_*` | arch-dispatch hooks | per-arch `RawSpinLock::arch_*` |

## Compatibility contract

REQ-1: `spinlock_t` layout:
- Embedded `raw_spinlock_t rlock` (the actual arch-lock).
- `struct lockdep_map dep_map` (lockdep tracking).
- `unsigned long magic` + `unsigned int owner_cpu` (debug build).

REQ-2: `raw_spinlock_t` layout:
- arch-spinlock data (qspinlock on x86_64: 32-bit value).
- `unsigned int magic` + `unsigned int owner_cpu` + `void *owner` (debug build).
- `struct lockdep_map dep_map`.

REQ-3: spin_lock semantics:
1. preempt_disable.
2. lockdep entry (LOCKDEP_RECURSION_LIMIT check + ordering record).
3. arch-spin-lock (qspinlock acquire).

REQ-4: spin_lock_irqsave semantics:
1. flags := local_irq_save (disable IRQ + save EFLAGS).
2. preempt_disable.
3. lockdep entry.
4. arch-spin-lock.

REQ-5: spin_lock_bh semantics:
1. local_bh_disable (atomically increments per-CPU softirq-disable counter).
2. preempt_disable (implicit via bh-disable).
3. lockdep entry.
4. arch-spin-lock.

REQ-6: PREEMPT_RT remap:
- `spinlock_t` becomes `rt_mutex_t` (sleepable PI mutex).
- `raw_spinlock_t` retains true-spin semantics; reserved for atomic-context (irq-disable, scheduler).
- Per-driver code that needs guarantees-no-sleep uses `raw_spinlock_t`; everything else uses `spinlock_t` for RT-friendliness.

REQ-7: Lockdep integration:
- Per-spinlock `lock_class_key` static-allocated at definition site.
- On lock acquire: lockdep records (subclass, ordering vs other locks held).
- Cycle-detection: depth-first search of lock-acquire graph; report deadlock if cycle.
- Recursion-detection: per-task depth counter capped at LOCKDEP_RECURSION_LIMIT.

REQ-8: Lockstat (CONFIG_LOCK_STAT):
- Per-lock-class statistics: per-CPU contention-count, holdtime histogram.
- /proc/lock_stat dump.

REQ-9: spin_unlock release barrier:
- arch-spin-unlock + preempt_enable.
- spin_unlock_irqrestore: arch-spin-unlock + local_irq_restore + preempt_enable.

REQ-10: Per-arch fallback (UP build):
- spinlock-on-UP becomes preempt_disable/enable only (no actual atomic).
- raw_spinlock-on-UP same.

REQ-11: Lockdep stress (lock_torture):
- /proc/lock_torture validates spinlock + rwlock + rt_mutex correctness under stress.

REQ-12: spin_dump / spin_bug debug-build:
- On detect-corrupt-lock or owner-cpu-mismatch: print stack trace + saved owner.

## Acceptance Criteria

- [ ] AC-1: Basic acquire-release: 1M lock+unlock cycles single-threaded < 50ms.
- [ ] AC-2: Multi-CPU contention: 16 CPUs spinning on same spinlock; throughput scales with qspinlock fairness; no starvation.
- [ ] AC-3: spin_lock_irqsave: while-held local-IRQ stays disabled; IRQ delivery delayed until unlock.
- [ ] AC-4: spin_trylock: successful path returns 1 + holds lock; failure path returns 0 + does not preempt-disable.
- [ ] AC-5: lockdep AB-BA detect: deliberate AB-BA acquire-order in test driver triggers lockdep splat with cycle trace.
- [ ] AC-6: PREEMPT_RT remap: same code path under PREEMPT_RT becomes rt_mutex; sleeping inside spin_lock(spinlock) section allowed; raw_spin_lock(raw) prohibited.
- [ ] AC-7: lock_torture stress 1h pass.
- [ ] AC-8: spinlock-debug variants: spin_dump on corrupted lock prints owner + stack.

## Architecture

`SpinLock<T>` is a typed RAII wrapper around the C-side spinlock_t:

```
struct SpinLock<T> {
  raw: RawSpinLock,
  data: UnsafeCell<T>,
}

struct RawSpinLock {
  arch_lock: ArchSpinLock,            // qspinlock 32-bit on x86_64
  // debug fields:
  magic: u32,
  owner_cpu: u32,
  owner: KAtomicPtr<TaskStruct>,
  dep_map: LockdepMap,
}

struct SpinLockGuard<'a, T> {
  lock: &'a SpinLock<T>,
  preempt_dec: bool,
  bh_dec: bool,
  irq_flags: Option<u64>,
}
```

`SpinLock::lock` flow:
1. preempt_disable.
2. spinlock_check (LOCKDEP_RECURSION_LIMIT + lockdep_acquire).
3. arch_spin_lock(&self.raw.arch_lock).
4. Set debug owner: this.raw.owner = current_task; this.raw.owner_cpu = smp_processor_id().
5. Return SpinLockGuard wrapping &mut self.data.

`SpinLockGuard::drop` (auto-unlock):
1. Clear debug owner.
2. lockdep_release.
3. arch_spin_unlock(&self.raw.arch_lock).
4. preempt_enable.

`SpinLock::lock_irqsave` flow:
1. flags := local_irq_save.
2. preempt_disable.
3. spinlock_check.
4. arch_spin_lock.
5. Return guard with irq_flags = Some(flags).

`SpinLock::lock_bh` flow:
1. local_bh_disable.
2. preempt_disable (implicit).
3. spinlock_check.
4. arch_spin_lock.

`SpinLock::try_lock`:
1. preempt_disable.
2. lockdep_try_acquire.
3. acquired := arch_spin_trylock(&self.raw.arch_lock).
4. If acquired: return Some(guard).
5. Else: preempt_enable; lockdep_release_try; return None.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lock_excludes_writers` | INVARIANT | at most one task holds RawSpinLock at any time. |
| `preempt_count_balanced` | INVARIANT | per-task preempt_count balanced across lock+unlock; defense against orphan preempt-disable. |
| `irq_flags_restored` | INVARIANT | per-irqsave guard restores exact saved EFLAGS at unlock; defense against IRQ-state leak. |
| `lockdep_recursion_capped` | INVARIANT | per-task lockdep depth ≤ LOCKDEP_RECURSION_LIMIT (typically 48). |

### Layer 2: TLA+

`kernel/locking/spinlock_acquire.tla`:
- States: Free, Held(owner_cpu).
- Transitions:
  - Free → Held(cpu) via lock; arch-CAS.
  - Held(cpu) → Free via unlock; release-store.
- Properties:
  - `safety_mutual_exclusion` — at most one cpu in Held at a time.
  - `liveness_held_eventually_free` — every Held eventually transitions to Free assuming no infinite loop in critical section.

`kernel/locking/qspinlock_mcs.tla` (already exists from earlier Tier-3 collection) covers MCS-queue fairness underlying qspinlock.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SpinLock::lock` post: arch_lock held; preempt count incremented; lockdep depth +1 | `SpinLock::lock` |
| `SpinLockGuard::drop` post: arch_lock free; preempt count decremented; lockdep depth -1 | `SpinLockGuard::drop` |
| `SpinLock::lock_irqsave` post: irq_flags set; arch_lock held | `SpinLock::lock_irqsave` |
| `SpinLock::try_lock` post: returns Some only if CAS succeeded | `SpinLock::try_lock` |
| Per-(lock_class, subclass) ordering acyclic via lockdep | `LockdepMap::acquire` |

### Layer 4: Verus/Creusot functional

`SpinLock::lock + critical section + drop guarantees mutual exclusion`: per-thread observation order ensures atomic visibility of inner-data updates only after lock-held period.

## Hardening

(Inherits row-1 features from `kernel/locking/00-overview.md` § Hardening.)

spinlock-specific reinforcement:

- **lockdep AB-BA cycle detection** — defense against ordering deadlock.
- **LOCKDEP_RECURSION_LIMIT** — defense against runaway lock-recursion.
- **debug owner_cpu validation** — defense against unlock-from-different-cpu corruption (CPU migration mid-critical-section).
- **spin_dump on corrupt magic** — defense against memory-corruption clobbering spinlock state.
- **Per-spinlock lock_class_key static** — defense against lockdep state leaking across instances.
- **PREEMPT_RT raw_spinlock_t remains true-spin** — defense against rt-mutex sleeping in atomic-context.
- **spin_lock_bh disables softirq atomically** — defense against softirq fire mid-critical-section reentering same lock.
- **arch_spin_lock memory-barriers** — defense against compiler/CPU reorder leaking pre-lock load past acquire.
- **Per-CPU preempt_count balance** — defense against orphan preempt-disable causing scheduler livelock.
- **irq_flags saved exactly via local_irq_save** — defense against torn EFLAGS read causing unrestored TF/AC bits.
- **Lockstat overhead gated by CONFIG_LOCK_STAT** — defense against per-acquire stat overhead in production.
- **lock_torture as default boot-time test option** — early detect spinlock breakage in CI.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on task/cred/sched_attr buffers.
- **PAX_KERNEXEC** — W^X for BPF JIT'd code, kprobe/uprobe trampolines.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization (critical for raw_spinlock paths invoked from syscall entry).
- **PAX_REFCOUNT** — saturating refcount on task_struct, files_struct, cred, mm_struct.
- **PAX_MEMORY_SANITIZE** — zero-on-free for task_struct, signal_struct, cred.
- **PAX_MEMORY_STACKLEAK** — kernel-stack zeroing on syscall exit; clears spinlock-debug owner stack frames leaked into reusable stack regions.
- **PAX_UDEREF** — strict user-pointer access for all task-pointer ops.
- **PAX_RAP / kCFI** — indirect-call signature enforcement for arch_spin_lock function pointers and lockdep callbacks.
- **GRKERNSEC_HIDESYM** — hide kernel addresses in /proc/<pid>/* + kallsyms; suppresses spinlock owner pointers in /proc/lock_stat.
- **GRKERNSEC_HARDEN_PTRACE** — restrict ptrace cross-uid (Yama scope ≥ 1); prevents attacker-controlled task inspecting in-flight spinlock state.
- **GRKERNSEC_BRUTE** — exponential delay on consecutive brute attempts (slows spin_lock_irqsave timing-channel probes).
- **GRKERNSEC_KSTACKOVERFLOW** — kernel-stack overflow guard against deep nested raw_spin_lock recursion.
- **GRKERNSEC_DMESG** — restrict syslog so lockdep splats leaking kernel pointers are unreadable to unprivileged users.
- **GRKERNSEC_SYSCTL_DISABLE** — disable dangerous sysctls (e.g. /proc/sys/kernel/lock_stat) by default.
- **GRKERNSEC_CONFIG_AUDIT** — boot-time runtime-config integrity check; verifies LOCKDEP/LOCK_STAT match expected production profile.
- **spin_dump panic-on-corrupt mode** — when GRKERNSEC_PANIC_ON_CORRUPTION is set, magic mismatch escalates to oops rather than warning.

Per-doc rationale: spinlocks are the lowest-level mutual-exclusion primitive and run in atomic/IRQ context where compromised state corrupts arbitrary task accounting; PaX REFCOUNT + KERNEXEC + RAP harden the lockdep/owner metadata and arch dispatch pointers, while Grsecurity HIDESYM/DMESG eliminates the unprivileged information leak channels that vanilla lockdep would otherwise expose.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- qspinlock arch backend (covered in `qspinlock.md` Tier-3)
- mutex (covered in `mutex.md` Tier-3)
- ww_mutex (covered in `mutex.md` Tier-3)
- rwlock_t reader-writer (covered in `kernel/locking/rwlock.md` future Tier-3)
- rt_mutex (covered in `kernel/locking/rtmutex.md` future Tier-3)
- semaphore (covered in `kernel/locking/semaphore.md` future Tier-3)
- percpu_rwsem (covered in `kernel/locking/percpu_rwsem.md` future Tier-3)
- Implementation code
