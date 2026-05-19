# Tier-3: kernel/locking/mutex.c — mutex + ww_mutex (sleep-allowed lock with optimistic spin + handoff + wound-wait)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/locking/00-overview.md
upstream-paths:
  - kernel/locking/mutex.c
  - kernel/locking/mutex.h
  - kernel/locking/ww_mutex.h
  - kernel/locking/mutex-debug.c
  - kernel/locking/osq_lock.c
  - include/linux/mutex.h
  - include/linux/ww_mutex.h
-->

## Summary

`mutex` is the universal sleep-allowed lock in the kernel — used by every kernel data structure that may be held across an operation that could sleep (memory allocation, IO submission, cross-CPU IPI). Replaces semaphores for binary mutual-exclusion since 2.6.16. Built-in optimistic-spinning fast path for short critical sections (avoids context switch when owner is currently running on another CPU); falls back to MCS-style queue + sleep + per-task wakeup for long contention. Owner-bit + handoff-bit + wait-bit + flags-bits all packed into a single 8-byte `owner` field for fast cmpxchg.

`ww_mutex` (wound-wait mutex) extends `mutex` for the use case of acquiring multiple mutexes in some order — necessary for DRM atomic-modeset locking N CRTC + N plane mutexes, GEM reservation chain locking. Wound-wait deadlock-avoidance: when a younger task tries to acquire a mutex held by an older task that's already trying to acquire a mutex held by the younger, the younger gets "wounded" (-EDEADLK) and must release all + retry from scratch via `ww_mutex_lock_slow`.

This Tier-3 covers `kernel/locking/mutex.c` (~1250 lines) + headers + osq_lock.c (per-CPU MCS spin lock used by mutex's optimistic-spin fast path).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mutex` | mutex control block (8 bytes: atomic owner+flags) | `kernel::sync::Mutex` |
| `struct mutex_waiter` | per-waiter list entry | `kernel::sync::MutexWaiter` |
| `struct ww_acquire_ctx` | per-ww-acquire context (per-task ticket + class) | `kernel::sync::WwAcquireCtx` |
| `struct ww_class` | per-ww-mutex class (acquire-context type tag) | `kernel::sync::WwClass` |
| `struct ww_mutex` | ww-mutex control block (mutex + ctx field) | `kernel::sync::WwMutex` |
| `mutex_init(&lock)` | init mutex | `Mutex::new` |
| `mutex_destroy(&lock)` | sanity-check before drop | (Drop impl asserts unlocked) |
| `mutex_lock(&lock)` / `_interruptible` / `_killable` | acquire | `Mutex::lock` / `_lock_interruptible` / `_lock_killable` |
| `mutex_unlock(&lock)` | release | `MutexGuard::drop` |
| `mutex_trylock(&lock)` | try-acquire | `Mutex::try_lock` |
| `mutex_is_locked(&lock)` | check held | `Mutex::is_locked` |
| `__mutex_lock(lock, state, subclass, nest_lock, ip, ww_ctx, use_ww_ctx)` | core slow-path | `Mutex::__lock` |
| `mutex_optimistic_spin(lock, ww_ctx, &waiter)` | optimistic-spin while owner runs on another CPU | `Mutex::optimistic_spin` |
| `__mutex_unlock_slowpath(lock, nested)` | wakeup-waiters slow-path | `Mutex::__unlock_slowpath` |
| `__mutex_handoff(lock, task)` | hand off to specific waiter | `Mutex::__handoff` |
| `__mutex_owner(lock)` | extract owner ptr from packed field | `Mutex::owner` |
| `__mutex_set_flag(lock, flag)` / `_clear_flag(...)` / `_trylock_or_owner(...)` / `_trylock_handoff(...)` | flag ops + try-acquire variants | `Mutex::__set_flag` / `_clear_flag` / etc. |
| `osq_lock(&lock)` / `_unlock(&lock)` | per-CPU MCS optimistic-spin queue | `OsqLock::lock` / `_unlock` |
| `ww_acquire_init(&ctx, &class)` / `_done(&ctx)` / `_fini(&ctx)` | per-acquire context lifecycle | `WwAcquireCtx::init` / `_done` / `_fini` |
| `ww_mutex_lock(&lock, &ctx)` / `_lock_interruptible(...)` | ww-acquire (may return -EDEADLK) | `WwMutex::lock` / `_lock_interruptible` |
| `ww_mutex_lock_slow(&lock, &ctx)` / `_lock_slow_interruptible(...)` | post-EDEADLK retry (blocking on contended lock) | `WwMutex::lock_slow` / `_lock_slow_interruptible` |
| `ww_mutex_unlock(&lock)` | release | `WwMutexGuard::drop` |
| `ww_mutex_trylock(&lock, &ctx)` | try-acquire | `WwMutex::try_lock` |
| `__ww_mutex_check_kill(lock, waiter, ww_ctx)` | wound check | `WwMutex::check_kill` |

## Compatibility contract

REQ-1: `struct mutex` is exactly 8 bytes (atomic owner+flags) on 64-bit (16 bytes incl debug + lockdep+spinwait extensions); preserves layout for embedded mutex in struct.

REQ-2: `mutex_lock(&lock)` semantics:
- If uncontended (owner == 0): single cmpxchg from 0 → current_task_ptr; succeeds; return.
- Else if owner running on another CPU: optimistic-spin via `osq_lock` + spin while owner is on-CPU.
- Else: full slow-path — add `mutex_waiter` to wait_list, call `__schedule(false)` to sleep until handoff.

REQ-3: `mutex_unlock(&lock)`:
- If wait_list empty: cmpxchg owner from current → 0; succeed; return.
- Else: `__mutex_unlock_slowpath` — wake first waiter; if `MUTEX_FLAG_HANDOFF` set, hand off lock directly to that waiter (avoid steal by spinning task).

REQ-4: Per-mutex packed flags: `MUTEX_FLAG_WAITERS` (set when wait_list non-empty), `MUTEX_FLAG_HANDOFF` (set when long-waiting first-waiter signals "give it to me directly"), `MUTEX_FLAG_PICKUP` (set transiently during handoff).

REQ-5: Optimistic-spin via per-CPU OSQ (Optimistic Spin Queue): MCS-style queue avoids cacheline ping-pong while waiters spin; spin only while owner is on-CPU AND wait_list empty AND TIF_NEED_RESCHED clear.

REQ-6: `mutex_lock_interruptible` returns -EINTR if signal arrives while sleeping; `mutex_lock_killable` returns -EINTR only on fatal signal (SIGKILL).

REQ-7: ww_mutex acquire-context (per-task per-class): unique ticket; younger ticket = larger value. Lock ordering: any ww_mutex held without a ctx is a deadlock-class violation.

REQ-8: ww_mutex wound-wait protocol:
- ww_mutex_lock(lock, ctx) blocks if contended; if blocking thread is older (smaller ticket) than current owner's ctx: success after wait.
- If blocking thread is younger (larger ticket) than current owner's ctx: -EDEADLK returned immediately to younger; younger MUST release all ww-locks + ww_acquire_done(ctx) + retry via ww_mutex_lock_slow on the contended lock.
- ww_mutex_lock_slow blocks unconditionally (no -EDEADLK return); used after backoff to ensure progress.

REQ-9: ww_acquire_done called after all ww-locks acquired + signals ctx is "done acquiring" — subsequent ww_mutex_lock with this ctx returns -EALREADY (alreadyholding).

REQ-10: Lockdep + lock-events tracepoint hooks preserved (per-acquire / per-release tracepoints; per-mutex deadlock detector).

## Acceptance Criteria

- [ ] AC-1: Microbenchmark uncontended `mutex_lock`/`_unlock` pair within 5% of upstream baseline.
- [ ] AC-2: Contention test: 64 threads contending single mutex; throughput within 5% upstream; no starvation observed via per-thread acquire-count balance.
- [ ] AC-3: Optimistic-spin test: 2 threads, brief critical section; verify via tracepoint that contention resolved in spin-path (no slow-path entered).
- [ ] AC-4: Interruptible mutex test: signal delivered while sleeping in `mutex_lock_interruptible` returns -EINTR; mutex still acquirable after signal handled.
- [ ] AC-5: ww_mutex chain test: 4 threads each acquire 4 mutexes in different orders; younger threads receive -EDEADLK; lock_slow retry converges; eventually all complete.
- [ ] AC-6: DRM atomic commit stress: concurrent commits referencing overlapping CRTC+plane ww-mutex sets converge correctly via wound-wait (cross-ref `drm-atomic.md`).
- [ ] AC-7: locktorture mutex stress (100s sustained, N=64) passes; no missed wakeup, no deadlock, no UAF.
- [ ] AC-8: KUnit `kernel/locking/mutex_test.c` passes.

## Architecture

`Mutex` lives in `kernel::sync::Mutex`:

```
#[repr(C)]
struct Mutex {
  owner: AtomicU64,             // bit 0-2: flags (WAITERS | HANDOFF | PICKUP); bit 3-63: task ptr (PID-style)
  wait_lock: RawSpinLock,
  osq: OsqLock,                  // optimistic-spin per-CPU queue
  wait_list: ListHead<MutexWaiter>,
  // CONFIG_DEBUG_MUTEXES + CONFIG_LOCKDEP fields...
}
```

Owner field encoding: lower 3 bits = flags (`_FLAG_WAITERS = 0x01`, `_FLAG_HANDOFF = 0x02`, `_FLAG_PICKUP = 0x04`); upper 61 bits = `task_struct *` ptr (clear lower 3 bits for owner extraction).

Fast-path `Mutex::lock`:
```
let zero: u64 = 0;
let curr = current_task() | flags_to_keep;
if self.owner.compare_exchange(zero, curr, Acquire, Relaxed).is_ok() {
  return MutexGuard::new(self);
}
self.__lock(TASK_UNINTERRUPTIBLE, /* ww_ctx = None */)
```

Slow-path `Mutex::__lock(state, ww_ctx)`:
1. If `Mutex::optimistic_spin(self, ww_ctx, &waiter)` succeeds: return.
2. Else: enter wait_list:
   - Take `wait_lock`.
   - Allocate `MutexWaiter` on stack; init `task = current()`, `ww_ctx = ctx`.
   - Insert into wait_list (FIFO order).
   - Set MUTEX_FLAG_WAITERS bit.
   - Drop `wait_lock`.
3. Loop:
   - Take `wait_lock`.
   - Try `Mutex::__try_lock_or_owner` (cmpxchg + test if we're the head waiter or PICKUP set).
   - If success: clear our wait_list entry; release wait_lock; return.
   - Else: __set_current_state(state); drop wait_lock; `schedule()`; we'll be woken by unlock or signal.
4. After return, ww_mutex variant runs `__ww_mutex_check_kill` to check wound condition.

`Mutex::optimistic_spin(self, ww_ctx, &waiter)`:
1. Check preconditions: `mutex_can_spin_on_owner(self)` — owner is on-CPU AND no waiters AND no TIF_NEED_RESCHED on current.
2. `osq_lock(&self.osq)` — enter per-CPU MCS queue.
3. While owner running on-CPU AND no need_resched AND no waiters appearing:
   - Try `Mutex::__trylock_or_owner` cmpxchg.
   - If success: `osq_unlock(&self.osq)`; return true (acquired).
   - Else: `cpu_relax()` (PAUSE / WFE).
4. `osq_unlock(&self.osq)`.
5. Return false (fall through to slow-path).

Unlock `Mutex::unlock` / `MutexGuard::drop`:
1. Try fast path: cmpxchg(owner, current, 0). If success + no MUTEX_FLAG_WAITERS: return.
2. Else `__mutex_unlock_slowpath`:
   - Take wait_lock.
   - Pop first waiter from wait_list.
   - If MUTEX_FLAG_HANDOFF set: set MUTEX_FLAG_PICKUP + atomic_xchg(owner, waiter | flags) so waiter sees PICKUP on its side.
   - Else: clear owner; waiter will steal via __try_lock_or_owner.
   - Wake waiter task via `wake_q_add` (wake_up_q after dropping wait_lock for batched wake).
   - Drop wait_lock.

WW-mutex wound-wait `WwMutex::lock(lock, ctx)`:
1. Standard `Mutex::__lock(.., ww_ctx=Some(ctx))`.
2. Inside slow-path, when about to schedule(): `__ww_mutex_check_kill(lock, waiter, ww_ctx)`:
   - Compare current ctx ticket to current owner's ctx ticket.
   - If current is younger (larger ticket): mark waiter wounded; return -EDEADLK.
3. Caller responsibility: on -EDEADLK, release all ww-locks acquired so far via `ww_acquire_done` / `_fini` + retry from scratch.
4. `ww_mutex_lock_slow(lock, ctx)` is the post-EDEADLK variant — blocks unconditionally (won't return -EDEADLK again).

Wound-detection on owner side: when newly-acquired ww_mutex's `ww_ctx` is set, walk wait_list — for each waiter with younger ctx, mark wounded + set wakeup; younger waiters re-check kill on wakeup, return -EDEADLK to caller.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `owner_field_atomicity` | ATOMICITY | every owner-field update via cmpxchg or xchg; never torn read by spinner. |
| `waiter_no_uaf` | UAF | `MutexWaiter` lifetime tied to caller's stack frame; release before stack frame returns; UAF impossible because slow-path doesn't return until waiter is freed. |
| `wait_list_no_corruption` | LIST-INTEGRITY | wait_list operations under wait_lock; insert/remove/peek serialized. |
| `osq_lock_no_starvation` | LIVENESS | OSQ MCS queue ensures bounded fairness in optimistic-spin path. |

### Layer 2: TLA+

`models/locking/mutex_handoff.tla` (declared in parent `kernel/locking/00-overview.md`): proves mutex handoff protocol — every release matched by exactly one acquire (no double-acquire); HANDOFF flag prevents starvation by long-waiting first-waiter.
`models/locking/wwmutex_wound.tla`: proves ww-mutex wound-wait deadlock-avoidance — under any acquire order from N tasks, eventual liveness via -EDEADLK + retry.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Mutual exclusion: at any instant, at most one task holds the mutex (owner field's task ptr unique) | `Mutex::lock` / `_unlock` |
| `mutex_unlock` post: owner field cleared OR new owner is the handoff target | `Mutex::unlock` |
| `ww_acquire_done` post: subsequent `ww_mutex_lock` with this ctx returns -EALREADY | `WwAcquireCtx::done` |
| Per-ctx ticket monotonic: each acquire-context gets a unique ticket; younger contexts have larger tickets | `WwAcquireCtx::init` |

### Layer 4: Verus/Creusot functional

Mutex mutual-exclusion + liveness: at any instant, at most one task holds; every contender eventually acquires (bounded by N waiters ahead). Encoded as Verus invariant.
ww-mutex deadlock-freedom: any concurrent `ww_mutex_lock` sequence from N tasks converges (no infinite -EDEADLK loop) because younger tasks always back off.

## Hardening

(Inherits row-1 features from `kernel/locking/00-overview.md` § Hardening.)

mutex-specific reinforcement:

- **Owner field cmpxchg-protected** — every owner update uses cmpxchg or xchg; defense against torn-write race.
- **Wait_lock acquired in canonical order** — never cross-mutex deadlock between two mutex's wait_locks.
- **OSQ optimistic-spin bounded** — TIF_NEED_RESCHED check ensures spinner yields if scheduler needs CPU.
- **`mutex_destroy` debug check** — CONFIG_DEBUG_MUTEXES asserts mutex is unlocked + wait_list empty before free; defense against UAF on lock-still-held drop.
- **Lockdep + lock-events tracepoint hooks** preserved — defense against lock ordering bugs via lockdep deadlock detector.
- **ww-mutex per-class enforcement** — per-`ww_class` distinct; cross-class ww-mutex acquire under wrong ctx triggers WARN.
- **`ww_acquire_done` invariant** — caller must call before releasing all ww-locks; debug-assertion in `ww_acquire_fini`.
- **`mutex_lock_interruptible` signal-delivery** preserves wait_list integrity via `__set_task_state` + signal_pending check before schedule.
- **HANDOFF flag prevents starvation** — long-waiting first-waiter sets HANDOFF; subsequent unlocker hands directly without spin-steal.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on task/cred/sched_attr buffers; rejects user copies that overlap mutex_waiter stack frames.
- **PAX_KERNEXEC** — W^X for BPF JIT'd code, kprobe/uprobe trampolines invoked while a mutex is held.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization; obscures `mutex_waiter`-on-stack layout against targeted overwrite.
- **PAX_REFCOUNT** — saturating refcount on task_struct (the owner ptr packed in `owner` field), files_struct, cred, mm_struct.
- **PAX_MEMORY_SANITIZE** — zero-on-free for task_struct, signal_struct, cred; stale owner pointers cannot be resurrected via slab reuse.
- **PAX_MEMORY_STACKLEAK** — kernel-stack zeroing on syscall exit; wipes leaked `MutexWaiter` after slow-path return.
- **PAX_UDEREF** — strict user-pointer access for all task-pointer ops invoked under ww_ctx-bearing slow-paths.
- **PAX_RAP / kCFI** — indirect-call signature enforcement for OSQ + lockdep callback vtables; mutex slow-path indirect dispatch validated.
- **GRKERNSEC_HIDESYM** — hide kernel addresses in /proc/<pid>/* + kallsyms; suppresses owner_ptr leak via /proc/lock_stat.
- **GRKERNSEC_HARDEN_PTRACE** — restrict ptrace cross-uid (Yama scope ≥ 1); blocks attacker from reading the owner field of a victim's mutex via ptrace-peek.
- **GRKERNSEC_BRUTE** — exponential delay on consecutive brute attempts; throttles timing-channel probes of optimistic-spin window.
- **GRKERNSEC_KSTACKOVERFLOW** — kernel-stack overflow guard against deep nested ww_mutex retry loops.
- **GRKERNSEC_DMESG** — restrict syslog so wound-wait WARN traces leaking owner ctx pointers are unreadable to unprivileged users.
- **GRKERNSEC_SYSCTL_DISABLE** — disable dangerous sysctls (lock_stat, lockup_detector tunables) by default.
- **GRKERNSEC_CONFIG_AUDIT** — boot-time runtime-config integrity check; verifies DEBUG_MUTEXES/LOCKDEP profile matches signed config.

Per-doc rationale: mutexes carry a raw `task_struct *` in their `owner` field and may sleep arbitrarily long, making them prime targets for owner-pointer disclosure and stale-pointer reuse; PaX MEMORY_SANITIZE + REFCOUNT prevent slab-reuse aliasing of stale owners, RAP/kCFI hardens the OSQ + ww-mutex indirect-call surface, and Grsecurity HIDESYM/DMESG closes the information-leak channels that would otherwise expose owner identity through /proc/lock_stat and lockdep WARN output.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- spinlock + qspinlock (covered in `kernel/locking/qspinlock.md` Tier-3)
- rwsem (covered in `kernel/locking/rwsem.md` future Tier-3)
- qrwlock (covered in `kernel/locking/qrwlock.md` future Tier-3)
- seqlock (covered in `kernel/locking/seqlock.md` future Tier-3)
- lockdep (covered in `kernel/locking/lockdep.md` future Tier-3)
- 32-bit-only paths
- Implementation code
