# Tier-3: lib/lockref.c — locked-reference-count (lockless cmpxchg-or-spinlock combo)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/locking/00-overview.md
upstream-paths:
  - lib/lockref.c (~163 lines)
  - include/linux/lockref.h (~63 lines)
  - arch/x86/Kconfig (CONFIG_ARCH_USE_CMPXCHG_LOCKREF=y)
  - arch/arm64/Kconfig (CONFIG_ARCH_USE_CMPXCHG_LOCKREF=y)
-->

## Summary

`struct lockref` packs an `int count` reference counter together with a `spinlock_t lock` into a single 8-byte word (`aligned_u64 lock_count` overlay) so that *both* the lock state and the count can be observed and conditionally updated in a single `cmpxchg64`. On architectures where `CONFIG_ARCH_USE_CMPXCHG_LOCKREF=y` and `SPINLOCK_SIZE <= 4` (x86_64, arm64), `lockref_get` / `_get_not_zero` / `_get_not_dead` / `_put_return` / `_put_or_lock` first attempt a bounded `cmpxchg` retry loop — checking the embedded spinlock is unlocked via `arch_spin_value_unlocked(old.lock.rlock.raw_lock)` and atomically swapping in a count-modified copy. On a CAS miss they fall back to the slow path: `spin_lock(&lockref->lock)`, update `count`, `spin_unlock`. Per-`lockref_mark_dead` sets count = -128 under spinlock; per-`__lockref_is_dead` is `(int)count < 0`. The primary consumer is `struct dentry` (`d_lockref`): VFS hot-path `dget()` / `dput()` use lockref-fastpath to avoid the spinlock on the common increment/decrement when no one else is contending. Critical for: pathname-lookup scalability (RCU-walk → ref-walk transition), file open throughput, anything that touches dcache.

This Tier-3 covers `lib/lockref.c` (~163 lines) + `include/linux/lockref.h` (~63 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct lockref` | per-{spinlock + int} 8-byte word | `LockRef` |
| `lockref.lock_count` (aligned_u64) | per-overlay 64-bit view | `LockRef::lock_count` |
| `lockref.lock` (spinlock_t) | per-spinlock half | `LockRef::lock` |
| `lockref.count` (int) | per-refcount half | `LockRef::count` |
| `USE_CMPXCHG_LOCKREF` (compile-time macro) | per-arch fast-path gate | `LockRef::USE_CMPXCHG` |
| `CMPXCHG_LOOP(CODE, SUCCESS)` (macro) | per-bounded-CAS retry loop | `LockRef::cmpxchg_loop` |
| `lockref_init()` (inline) | per-init: count=1, spin_lock_init | `LockRef::init` |
| `lockref_get()` | per-unconditional ++count | `LockRef::get` |
| `lockref_get_not_zero()` | per-conditional ++count if count>0 | `LockRef::get_not_zero` |
| `lockref_get_not_dead()` | per-conditional ++count if count>=0 (not dead) | `LockRef::get_not_dead` |
| `lockref_put_return()` | per--count, return new count or -1 | `LockRef::put_return` |
| `lockref_put_or_lock()` | per--count if count>1 else acquire lock | `LockRef::put_or_lock` |
| `lockref_mark_dead()` | per-store count=-128 under held lock | `LockRef::mark_dead` |
| `__lockref_is_dead()` (inline) | per-test count<0 | `LockRef::is_dead` |
| `arch_spin_value_unlocked()` | per-lock-half unlocked check | shared spinlock-side |
| `try_cmpxchg64_relaxed()` | per-CAS primitive | shared atomic-side |
| `spin_lock()` / `spin_unlock()` | per-slowpath lock acquire | shared spinlock-side |
| `assert_spin_locked()` | per-mark_dead precondition | shared spinlock-side |

## Compatibility contract

REQ-1: struct lockref layout:
- union of:
  - `aligned_u64 lock_count` (when USE_CMPXCHG_LOCKREF).
  - anonymous struct { `spinlock_t lock; int count;` } (always).
- `BUILD_BUG_ON(sizeof(struct lockref) != 8)` enforced at compile time inside CMPXCHG_LOOP.
- Layout assumption: `lock` precedes `count`; little-endian arches put `lock` in low 32 bits.

REQ-2: USE_CMPXCHG_LOCKREF compile-time gate:
- `IS_ENABLED(CONFIG_ARCH_USE_CMPXCHG_LOCKREF) ∧ IS_ENABLED(CONFIG_SMP) ∧ SPINLOCK_SIZE <= 4`.
- x86_64 / arm64 / s390 / loongarch enable; ARM32 / RISC-V / 32-bit-x86 disable (qspinlock 32-bit aligned check satisfied only on selected arches).
- When disabled, `CMPXCHG_LOOP` expands to `do {} while (0)`: every API call takes the slow path.

REQ-3: lockref_init() (inline, in header):
- `spin_lock_init(&lockref->lock)`.
- `lockref->count = 1`.

REQ-4: lockref_get(lockref):
- /* Fast path */
- `CMPXCHG_LOOP(new.count++; , return;)`:
  - retry = 100.
  - `old.lock_count = READ_ONCE(lockref->lock_count)`.
  - while `arch_spin_value_unlocked(old.lock.rlock.raw_lock)`:
    - new = old; new.count++.
    - if `try_cmpxchg64_relaxed(&lockref->lock_count, &old.lock_count, new.lock_count)`: return.
    - if --retry == 0: break.
- /* Slow path */
- `spin_lock(&lockref->lock)`.
- `lockref->count++`.
- `spin_unlock(&lockref->lock)`.
- Precondition: caller already holds a reference (count cannot be 0 from caller's view).

REQ-5: lockref_get_not_zero(lockref) -> bool:
- /* Fast path */
- `CMPXCHG_LOOP(new.count++; if (old.count <= 0) return false; , return true;)`:
  - if CAS-eligible and old.count > 0 and CAS succeeds: return true.
  - if old.count <= 0 inside loop: return false (do not increment, do not take slow path).
- /* Slow path */
- retval = false.
- `spin_lock(&lockref->lock)`.
- if `lockref->count > 0`: `lockref->count++; retval = true`.
- `spin_unlock(&lockref->lock)`.
- return retval.

REQ-6: lockref_get_not_dead(lockref) -> bool:
- /* Fast path */
- `CMPXCHG_LOOP(new.count++; if (old.count < 0) return false; , return true;)`.
- /* Slow path */
- retval = false.
- `spin_lock(&lockref->lock)`.
- if `lockref->count >= 0`: `lockref->count++; retval = true`.
- `spin_unlock(&lockref->lock)`.
- return retval.

REQ-7: lockref_put_return(lockref) -> int:
- /* Fast path only */
- `CMPXCHG_LOOP(new.count--; if (old.count <= 0) return -1; , return new.count;)`.
- /* On CAS miss / retry-exhausted / disabled: return -1 (caller must use put_or_lock) */
- return -1.

REQ-8: lockref_put_or_lock(lockref) -> bool [conditionally-acquires &lockref->lock]:
- /* Fast path */
- `CMPXCHG_LOOP(new.count--; if (old.count <= 1) break; , return true;)`:
  - if old.count > 1 and CAS succeeds: return true (count decremented, lock NOT held).
  - if old.count <= 1: break out of loop (forces slow path).
- /* Slow path */
- `spin_lock(&lockref->lock)`.
- if `lockref->count <= 1`: return false (count NOT decremented, lock HELD — caller must release or free).
- `lockref->count--`.
- `spin_unlock(&lockref->lock)`.
- return true.
- Semantics: caller wants to drop a ref; if doing so would reach count==0 (death), keep the lock so caller can finalize.

REQ-9: lockref_mark_dead(lockref):
- `assert_spin_locked(&lockref->lock)` — caller must already hold the lock.
- `lockref->count = -128` (sentinel; any negative value indicates death; -128 chosen to be unambiguous and to give room for transient -1 returns).

REQ-10: __lockref_is_dead(l) -> bool (inline, header):
- "Must be called under spinlock for reliable results".
- return `((int)l->count < 0)`.
- Unsynchronized read is acceptable on the fast path: the value read may be stale but never spuriously claims a live ref as dead because a transition to dead requires acquiring the spinlock.

REQ-11: CMPXCHG_LOOP macro contract:
- Bounded by `retry = 100` (no unbounded spinning under livelock).
- `BUILD_BUG_ON(sizeof(old) != 8)` — abort compile if layout drift breaks the 8-byte invariant.
- Reads via `READ_ONCE` to prevent the compiler from re-loading mid-loop.
- Uses `try_cmpxchg64_relaxed` — no memory ordering beyond what the inner load already provides.
- CODE block executes against `old` and produces a `new` value; SUCCESS block runs on CAS-success.

REQ-12: Per-EXPORT list:
- `EXPORT_SYMBOL(lockref_get)` — non-GPL.
- `EXPORT_SYMBOL(lockref_get_not_zero)` — non-GPL.
- `EXPORT_SYMBOL(lockref_put_return)` — non-GPL.
- `EXPORT_SYMBOL(lockref_put_or_lock)` — non-GPL.
- `EXPORT_SYMBOL(lockref_mark_dead)` — non-GPL.
- `EXPORT_SYMBOL(lockref_get_not_dead)` — non-GPL.

REQ-13: Per-consumer set (canonical):
- `struct dentry::d_lockref` — VFS dcache reference counting:
  - `dget()` → `lockref_get(&dentry->d_lockref)`.
  - `dput()` → `lockref_put_return(&dentry->d_lockref)` or `_put_or_lock(&dentry->d_lockref)`.
  - `__d_drop()` / `__dentry_kill()` → `lockref_mark_dead(&dentry->d_lockref)` under d_lock.
  - `lockref_get_not_dead` used in RCU-walk → ref-walk transition.

REQ-14: Memory ordering:
- `try_cmpxchg64_relaxed` — relaxed; no extra barrier.
- The "released" reference is published by the slow path's `spin_unlock` (release semantics) when contention forces it.
- The "acquired" reference is implicitly synchronized via the data the caller is about to access — caller code is responsible for any further barriers (e.g. `READ_ONCE`/`smp_rmb`).
- Fast-path CAS gives no ordering — adequate because the fast path only mutates the count, never reads dependent data.

REQ-15: Failure modes:
- `lockref_put_return` returning -1: caller MUST treat as either "lock contention" or "would-be-zero" and fall back to `lockref_put_or_lock`.
- `lockref_get_not_zero` / `_get_not_dead` returning false: object is dying / dead; caller MUST NOT retry without re-acquiring a stronger reference (e.g. parent's lock).
- `lockref_put_or_lock` returning false: lock is HELD; caller MUST eventually `spin_unlock` (typically via `__dentry_kill` path).

## Acceptance Criteria

- [ ] AC-1: `sizeof(struct lockref) == 8` on all configured arches; verified by BUILD_BUG_ON inside CMPXCHG_LOOP.
- [ ] AC-2: `lockref_init`: count==1, lock unlocked.
- [ ] AC-3: `lockref_get` on uncontended unlocked lockref: increments count via CAS, never acquires lock.
- [ ] AC-4: `lockref_get` on contended (lock held by another CPU) lockref: falls back to `spin_lock`/`++`/`spin_unlock`.
- [ ] AC-5: `lockref_get_not_zero` on count==0: returns false; count unchanged.
- [ ] AC-6: `lockref_get_not_dead` on count==-128: returns false; count unchanged.
- [ ] AC-7: `lockref_put_return` decrements and returns new count when count was > 1 before and CAS succeeded.
- [ ] AC-8: `lockref_put_return` returns -1 when count <= 0 was observed in fast path or USE_CMPXCHG_LOCKREF=0.
- [ ] AC-9: `lockref_put_or_lock` on count > 1: decrements, returns true, lock NOT held on return.
- [ ] AC-10: `lockref_put_or_lock` on count == 1: returns false, lock IS held on return, count unchanged.
- [ ] AC-11: `lockref_mark_dead`: count == -128 after; precondition asserts spinlock held.
- [ ] AC-12: CMPXCHG_LOOP bounded by retry=100; livelock impossible.
- [ ] AC-13: When USE_CMPXCHG_LOCKREF=0, every API call goes through the spinlock slow path.
- [ ] AC-14: On x86_64 (USE_CMPXCHG_LOCKREF=1), uncontended `lockref_get`/`_put_return` issue exactly one `cmpxchg` (no spinlock instructions).

## Architecture

```
struct LockRef {
    // Conceptual layout — 8 bytes total.
    // On USE_CMPXCHG arches, a u64 union overlay is used:
    //   lock_count: AlignedU64,
    // else, only the (lock, count) fields are visible.
    lock: SpinLock,    // raw_spinlock_t inner, 4 bytes
    count: i32,        // signed; negative = dead
}
```

`LockRef::USE_CMPXCHG: bool` — compile-time const:
- `cfg!(ARCH_USE_CMPXCHG_LOCKREF) && cfg!(SMP) && SPINLOCK_SIZE_BYTES <= 4`.

`LockRef::init(self)`:
1. `SpinLock::init(&mut self.lock)`.
2. self.count = 1.

`LockRef::cmpxchg_loop<F, S>(self, code: F, success: S) -> ControlFlow`:
1. const RETRY: usize = 100.
2. if !USE_CMPXCHG: return ControlFlow::FellThrough.
3. let mut old: LockRef = read_once(self).
4. const_assert!(size_of::<LockRef>() == 8).
5. for _ in 0..RETRY:
   - if !arch_spin_value_unlocked(&old.lock.raw): return ControlFlow::FellThrough.
   - let mut new = old; code(&mut new, &old) — may early-return ControlFlow::Done(value).
   - if try_cmpxchg64_relaxed(&self.lock_count, &mut old.lock_count, new.lock_count): return success(new).
6. return ControlFlow::FellThrough.

`LockRef::get(&self)`:
1. `cmpxchg_loop(|new, _old| new.count += 1, |_new| return)`.
2. /* Slow path */
3. self.lock.lock().
4. self.count += 1.
5. self.lock.unlock().

`LockRef::get_not_zero(&self) -> bool`:
1. `cmpxchg_loop(|new, old| { new.count += 1; if old.count <= 0 return Done(false) }, |_new| return true)`.
2. /* Slow path */
3. let mut retval = false.
4. self.lock.lock().
5. if self.count > 0: self.count += 1; retval = true.
6. self.lock.unlock().
7. return retval.

`LockRef::get_not_dead(&self) -> bool`:
1. `cmpxchg_loop(|new, old| { new.count += 1; if old.count < 0 return Done(false) }, |_new| return true)`.
2. /* Slow path */
3. let mut retval = false.
4. self.lock.lock().
5. if self.count >= 0: self.count += 1; retval = true.
6. self.lock.unlock().
7. return retval.

`LockRef::put_return(&self) -> i32`:
1. `cmpxchg_loop(|new, old| { new.count -= 1; if old.count <= 0 return Done(-1) }, |new| return new.count)`.
2. /* No slow path — fall-through returns -1 */.
3. return -1.

`LockRef::put_or_lock(&self) -> bool` (conditionally-acquires `&self.lock`):
1. `cmpxchg_loop(|new, old| { new.count -= 1; if old.count <= 1 break }, |_new| return true)`.
2. /* Slow path */
3. self.lock.lock().
4. if self.count <= 1: return false (LOCK HELD).
5. self.count -= 1.
6. self.lock.unlock().
7. return true.

`LockRef::mark_dead(&mut self)`:
1. debug_assert!(self.lock.is_held()).
2. self.count = -128.

`LockRef::is_dead(&self) -> bool` (inline; caller must hold lock for reliable result):
1. return (self.count as i32) < 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sizeof_lockref_is_8` | INVARIANT | per-LockRef: size_of::<LockRef>() == 8. |
| `mark_dead_requires_lock` | INVARIANT | per-mark_dead: lock held precondition asserted. |
| `cmpxchg_loop_bounded` | INVARIANT | per-cmpxchg_loop: retries <= 100; no unbounded spin. |
| `get_not_zero_no_resurrect` | INVARIANT | per-get_not_zero: count==0 ⟹ returns false; count unchanged. |
| `get_not_dead_no_resurrect` | INVARIANT | per-get_not_dead: count<0 ⟹ returns false; count unchanged. |
| `put_or_lock_zero_keeps_lock` | INVARIANT | per-put_or_lock: returns false ⟹ lock held; count unchanged. |
| `put_or_lock_true_releases_lock` | INVARIANT | per-put_or_lock: returns true ⟹ lock released; count decremented. |
| `put_return_minus1_on_zero` | INVARIANT | per-put_return: count<=0 fast-path observation ⟹ -1. |
| `fastpath_no_spinlock_when_unlocked` | INVARIANT | per-cmpxchg fastpath: arch_spin_value_unlocked-gated; no spin_lock issued. |
| `slowpath_acquires_lock` | INVARIANT | per-fallback: spin_lock taken before count mutation. |
| `dead_sentinel_minus_128` | INVARIANT | per-mark_dead post: count == -128. |

### Layer 2: TLA+

`kernel/locking/lockref.tla`:
- Per-N-CPU model with M lockrefs, each modeled as (lock∈{LOCKED, UNLOCKED}, count∈Int).
- Actions: `Get(c, lr)`, `GetNotZero(c, lr)`, `PutReturn(c, lr)`, `PutOrLock(c, lr)`, `MarkDead(c, lr)`.
- Properties:
  - `safety_dead_never_resurrected` — per-MarkDead: count<0 holds forever after the action, until lockref is reinitialized.
  - `safety_count_monotonic_after_dead` — per-dead: no Get* action increments count.
  - `safety_put_or_lock_invariant` — per-PutOrLock: result==false ⟹ lock state LOCKED at return.
  - `safety_lockref_total_count_conservation` — per-(N gets vs M puts): final count == initial + N - M (modulo death).
  - `liveness_uncontended_fastpath_terminates` — per-no-contention: CAS succeeds within RETRY=100.
  - `liveness_contended_slowpath_terminates` — per-spin_lock fairness: every API caller eventually returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `LockRef::init` post: count==1, lock unlocked | `LockRef::init` |
| `LockRef::get` post: count == pre+1 (when caller already holds a ref) | `LockRef::get` |
| `LockRef::get_not_zero` post: result==true ⟹ count == pre+1 ∧ pre > 0; result==false ⟹ count == pre ∧ pre <= 0 | `LockRef::get_not_zero` |
| `LockRef::get_not_dead` post: result==true ⟹ count == pre+1 ∧ pre >= 0; result==false ⟹ count == pre ∧ pre < 0 | `LockRef::get_not_dead` |
| `LockRef::put_return` post: result == new_count when CAS succeeded; result == -1 otherwise | `LockRef::put_return` |
| `LockRef::put_or_lock` post: result==true ⟹ count == pre-1 ∧ pre > 1 ∧ lock released; result==false ⟹ count == pre ∧ pre <= 1 ∧ lock held | `LockRef::put_or_lock` |
| `LockRef::mark_dead` pre: lock held; post: count == -128 | `LockRef::mark_dead` |
| `LockRef::is_dead` post: result == (count < 0) at observation point | `LockRef::is_dead` |
| `LockRef::cmpxchg_loop` post: ControlFlow::FellThrough ⟹ no count or lock state change observable | `LockRef::cmpxchg_loop` |

### Layer 4: Verus/Creusot functional

`Per-lockref API sequence on a single lockref` semantic equivalence: for any concurrent program over lockref methods, observed (count, lock_state) sequences must be a refinement of the TLA+ abstract model. Per-Documentation/admin-guide/sysctl/vm.rst not applicable here; per-Documentation/filesystems/path-lookup.rst describes the dcache use; per-Documentation/atomic_t.txt describes the memory-ordering contract honored.

## Hardening

(Inherits row-1 features from `kernel/locking/00-overview.md` § Hardening.)

lockref reinforcement:

- **Per-`BUILD_BUG_ON(sizeof(struct lockref) != 8)`** — defense against per-layout-drift breaking the cmpxchg64 invariant.
- **Per-`retry = 100` bound** — defense against per-livelock under sustained CAS contention; deterministic fallback to spinlock.
- **Per-`arch_spin_value_unlocked` precondition before CAS** — defense against per-fastpath-mutating-during-mark_dead; the slow path holds the lock so the fast path sees a non-unlocked value and aborts cleanly.
- **Per-`READ_ONCE(lockref->lock_count)`** — defense against per-compiler-reload mid-loop that would skew (old, new) pairing.
- **Per-`try_cmpxchg64_relaxed`** — defense against per-overstrong-barriers degrading scalability (the contract intentionally provides no extra ordering).
- **Per-`-128` dead sentinel** — defense against per-confusing -1 returns from put_return; -128 is unambiguous (and still negative for the `(int)count < 0` test).
- **Per-`assert_spin_locked` in mark_dead** — defense against per-unlocked-mark-dead racing get/put.
- **Per-`put_or_lock` returning lock-held** — defense against per-double-finalize and per-use-after-free in dentry kill: caller can serialize via the held lock.
- **Per-`get_not_dead` for RCU-walk transition** — defense against per-acquiring-ref-on-already-dying dentry during RCU lookup.
- **Per-`USE_CMPXCHG_LOCKREF=0` slow-path fallback** — defense against per-arch-without-cmpxchg64 silently breaking semantics; the slow path is functionally identical.
- **Per-`EXPORT_SYMBOL` (not _GPL)** — intentional: legacy stable ABI for out-of-tree dcache consumers; Rookery should match.
- **Per-no-memory-barriers-in-fastpath** — defense against per-overcautious-stalls; caller code is responsible for any further ordering via standard atomic_t.txt rules.
- **Per-`__lockref_is_dead` inline + "call under spinlock" comment** — defense against per-unsynchronized-test producing actionable false negatives (a dying lockref might still read live, but a live lockref will never read dead because the transition happens under spinlock).

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy.
- **PAX_KERNEXEC** — W^X for any executable mapping.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization.
- **PAX_REFCOUNT** — saturating refcount on subsystem structs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for sensitive allocations.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on vtables.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding.
- **GRKERNSEC_DMESG** — syslog restriction.
- **CMPXCHG loop bounded iterations** — the `CMPXCHG_LOOP(...)` macro caps retries (LOCKREF_RETRY_COUNT); runaway contention from a malicious priority-inversion drops to the slow path under spinlock instead of looping unboundedly.
- **Dead-flag canonicalization** — `mark_dead` sets `count = -128`, a value PAX_REFCOUNT recognizes as poisoned; any subsequent `++` saturates rather than resurrecting the lockref to a live count.
- **Aligned `struct lockref` for atomic128** — alignment asserted at compile time so a misaligned attacker-influenced placement cannot split the cmpxchg into two non-atomic halves on x86-64.
- **No-userspace surface** — lockref has no /proc, /sys, or syscall consumer; PAX_USERCOPY scope is purely transitive through dcache callers.
- **Slow-path lock acquisition under PAX_RAP** — fallback `spin_lock + lockref_get_not_zero` indirect dispatch matches kCFI signature; gadgetized override of the fallback fails before touching `->count`.
- **EXPORT_SYMBOL hidden symbols** — under GRKERNSEC_HIDESYM, exported lockref helpers do not leak addresses through `/proc/kallsyms` to unprivileged readers.

Per-doc rationale: lockref's value to an attacker is the cmpxchg fastpath — a wrap from `-128` back to a positive count, or an unbounded retry against a victim CPU, would resurrect a freed dentry or pin a CPU. PAX_REFCOUNT saturation on the count, a bounded CMPXCHG_LOOP, alignment guarantees for the 128-bit CAS, and PAX_RAP on the slow-path indirect dispatch close these holes; GRKERNSEC_HIDESYM keeps the exported helpers off the kallsyms leak surface for unprivileged callers.

## Open Questions

- Future arch enablement (RISC-V with Zacas): USE_CMPXCHG_LOCKREF can be set; verify SPINLOCK_SIZE check still holds.
- PREEMPT_RT: `spinlock_t` becomes an rt_mutex. USE_CMPXCHG_LOCKREF should be effectively disabled because RT spinlocks are not arch-spinlocks — verify Rookery PREEMPT_RT lockref path defaults to slow.
- Should `lockref_get_not_dead` ever take the slow path? Upstream does (and may grow count to >=0 from -128 if a careless caller resurrects via mark-dead-then-init). Confirm Rookery API matches.
- Is `lockref_put_return` returning `new.count` from a stale `old.count` race possible? Per CMPXCHG_LOOP semantics, no — CAS ties `new` to a fresh `old`, so `new.count` is the actual post-state count.

## Out of Scope

- `kernel/locking/spinlock.md` — owns `spin_lock`, `spin_unlock`, `arch_spin_value_unlocked` semantics.
- `kernel/locking/qspinlock.md` — owns the queued-spinlock implementation underneath `spinlock_t`.
- `fs/dcache/00-overview.md` (if added) — owns `struct dentry::d_lockref` usage and dget/dput call sites.
- `include/linux/atomic.h` / `arch/x86/include/asm/cmpxchg_64.h` — owns `try_cmpxchg64_relaxed` primitive.
- PREEMPT_RT rt_mutex semantics — covered under `kernel/locking/rtmutex.md` if added.
- `lib/refcount.c` (saturating refcount_t) — different abstraction; covered in `lib/data-structures.md` or future `kernel/locking/refcount.md`.
- `lib/percpu-refcount.c` — different abstraction (percpu fast path); covered in future Tier-3 if added.
- Implementation code.
