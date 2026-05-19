# Tier-3: kernel/futex/pi.c — Priority Inheritance futex (FUTEX_LOCK_PI / UNLOCK_PI / TRYLOCK_PI / WAIT_REQUEUE_PI)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/futex/core.md
upstream-paths:
  - kernel/futex/pi.c (~1304 lines)
  - kernel/futex/futex.h
  - kernel/locking/rtmutex.c (PI rt_mutex backend)
  - include/uapi/linux/futex.h (FUTEX_LOCK_PI, FUTEX_UNLOCK_PI, ...)
-->

## Summary

PI-futexes provide priority inheritance for user-space mutex implementations. Per-`FUTEX_LOCK_PI`: kernel acquires `rt_mutex` proxy on user-space futex word; per-priority-boost propagates from waiter to current owner via PI-chain. Per-futex-word format: bits[0..29] = TID of owner; bit 30 = FUTEX_WAITERS; bit 31 = FUTEX_OWNER_DIED. Per-`FUTEX_UNLOCK_PI`: clears futex word + wakes top-waiter (highest priority). Per-`FUTEX_TRYLOCK_PI`: non-blocking attempt. Per-`FUTEX_CMP_REQUEUE_PI`: requeue from non-PI to PI-mutex. Per-PI propagates through nested rt_mutex chains. Critical for: pthread_mutex with PI, real-time apps, deadlock-free PI-protocol.

This Tier-3 covers `kernel/futex/pi.c` (~1304 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `futex_lock_pi()` | per-LOCK_PI | `Futex::lock_pi` |
| `futex_unlock_pi()` | per-UNLOCK_PI | `Futex::unlock_pi` |
| `futex_lock_pi_atomic()` | per-fast-path acquire | `Futex::lock_pi_atomic` |
| `wait_for_owner_exiting()` | per-OWNER_DIED | `Futex::wait_for_owner_exiting` |
| `attach_to_pi_state()` | per-attach-existing | `Futex::attach_to_pi_state` |
| `attach_to_pi_owner()` | per-attach-new | `Futex::attach_to_pi_owner` |
| `lookup_pi_state()` | per-lookup | `Futex::lookup_pi_state` |
| `refill_pi_state_cache()` | per-cache | `Futex::refill_pi_state_cache` |
| `alloc_pi_state()` | per-alloc | `Futex::alloc_pi_state` |
| `free_pi_state()` | per-free | `Futex::free_pi_state` |
| `wake_futex_pi()` | per-wake-top | `Futex::wake_pi` |
| `fixup_owner()` / `fixup_pi_state_owner()` | per-rebind owner | `Futex::fixup_*_owner` |
| `requeue_pi_wake_futex()` | per-CMP_REQUEUE_PI wake | `Futex::requeue_pi_wake_futex` |
| `pi_state_update_owner()` | per-update | `Futex::pi_state_update_owner` |
| `handle_exit_race()` | per-owner-exit race | `Futex::handle_exit_race` |
| `task_pi_state` (per-task list) | per-task PI-state list | `Task::pi_state_list` |

## Compatibility contract

REQ-1: Futex word format (PI):
- bits[0..29] = TID of owner.
- bit 30 = FUTEX_WAITERS (waiters present).
- bit 31 = FUTEX_OWNER_DIED.
- 0 = unlocked.

REQ-2: futex_lock_pi(uaddr, flags, timeout):
- /* Validate */
- if !valid TID format: -EINVAL.
- /* Try fast-path: cmpxchg 0 → current_tid */
- ret = futex_lock_pi_atomic(uaddr, &q, &exiting).
- if ret == 0: return 0 (acquired).
- if ret == -EFAULT: handle_fault.
- /* Slow-path: enqueue on rt_mutex */
- queue_me(&q).
- ret = rt_mutex_timed_futex_lock(&pi_state.pi_mutex).
- /* On wakeup: fixup owner, return */
- fixup_pi_state_owner(...).
- unqueue_me(&q).
- return ret.

REQ-3: futex_lock_pi_atomic(uaddr, q, exiting):
- /* Read uval */
- ret = get_futex_value_locked(&uval, uaddr).
- if ret: return ret.
- ownertid = uval & FUTEX_TID_MASK.
- if ownertid == 0:
  - /* Try acquire */
  - ret = atomic_cmpxchg(uaddr, 0, current.pid).
  - if ret == 0: return 1 (acquired).
- /* Owner exists */
- if uval & FUTEX_OWNER_DIED: handle_owner_died.
- attach_to_pi_owner / attach_to_pi_state.
- return 0 (queued for slow-path).

REQ-4: attach_to_pi_owner(uval, uaddr, key, ...):
- /* Find owner-task by TID */
- p = find_get_task_by_vpid(ownertid).
- if !p: return -ESRCH.
- /* Avoid attach if owner is exiting (race) */
- if p.flags & PF_EXITING:
  - handle_exit_race; return retry.
- /* Allocate pi_state */
- pi_state = alloc_pi_state.
- pi_state.owner = p.
- pi_state.key = *key.
- /* Init rt_mutex */
- rt_mutex_init_proxy_locked(&pi_state.pi_mutex, p).
- /* Add to task's pi-state list */
- list_add(&pi_state.list, &p.pi_state_list).
- put_pi_state.

REQ-5: futex_unlock_pi(uaddr, flags):
- /* Read uval */
- get_futex_value_locked(&uval, uaddr).
- vpid = task_pid_vnr(current).
- if (uval & FUTEX_TID_MASK) != vpid: return -EPERM.
- /* No waiters: clear */
- if !(uval & FUTEX_WAITERS):
  - cmpxchg uaddr (uval) → 0.
- /* Waiters: pass to top */
- top_waiter = pi_state_top_waiter.
- new_owner = top_waiter.task.
- /* Atomically set futex_word to new_owner | WAITERS-if-more */
- cmpxchg uaddr → new_owner | (multiple_waiters ? WAITERS : 0).
- /* Wake top */
- wake_futex_pi(&top_waiter).

REQ-6: futex_trylock_pi(uaddr, flags):
- ret = lock_pi_atomic(uaddr, ...).
- if ret == 1: return 0 (acquired).
- /* Blocked → release pi_state and return -EBUSY */
- return -EBUSY.

REQ-7: FUTEX_CMP_REQUEUE_PI:
- Per-cond-var pthread_cond_signal: requeue waiters from non-PI to PI mutex.
- requeue_pi_wake_futex(&q, &key2, &hb2).

REQ-8: rt_mutex backend:
- pi_state.pi_mutex: rt_mutex.
- rt_mutex_setprio: per-PI propagation through chain.
- rt_mutex_top_waiter.

REQ-9: Per-task pi_state_list:
- task.pi_state_list: list of pi_states this task owns.
- exit_pi_state_list at task-exit: walks list, marks each FUTEX_OWNER_DIED.

REQ-10: FUTEX_OWNER_DIED:
- Set by exit_pi_state_list when owner dies.
- Next FUTEX_LOCK_PI: handle as cleanup-needed.

REQ-11: PI-state cache:
- per-CPU pi_state_cache for fast alloc.
- refill on slow path.

REQ-12: fixup_pi_state_owner / fixup_owner:
- Post-rt_mutex release: ensure futex word + pi_state.owner consistent.
- May be necessary if waiter was preempted between rt_mutex_release and futex-word-update.

REQ-13: ABA / robust-futex interaction:
- Robust-futex list: per-task list of held PI-mutexes.
- exit_robust_list at task-exit: cleans futex words (sets FUTEX_OWNER_DIED).

REQ-14: Per-priority propagation:
- waiter.prio > owner.prio: rt_mutex_setprio(owner, waiter.prio).
- Chain-walks through nested rt_mutex acquisitions.

REQ-15: Deadlock detection:
- rt_mutex chain-walk detects cycles → -EDEADLK.

REQ-16: Real-time scheduling:
- PI works for SCHED_FIFO / SCHED_RR / SCHED_DEADLINE.

## Acceptance Criteria

- [ ] AC-1: FUTEX_LOCK_PI on free futex: fast-path cmpxchg succeeds; word = TID.
- [ ] AC-2: FUTEX_LOCK_PI on owned futex: pi_state allocated; word |= WAITERS.
- [ ] AC-3: Higher-prio waiter: owner's prio boosted via rt_mutex.
- [ ] AC-4: FUTEX_UNLOCK_PI no waiters: word = 0.
- [ ] AC-5: FUTEX_UNLOCK_PI with waiters: word = top-waiter-TID; top wakes.
- [ ] AC-6: Owner exits while held: FUTEX_OWNER_DIED set; next acquirer cleans.
- [ ] AC-7: FUTEX_TRYLOCK_PI on owned: -EBUSY.
- [ ] AC-8: FUTEX_CMP_REQUEUE_PI: requeue from non-PI to PI mutex.
- [ ] AC-9: Deadlock: rt_mutex chain-walk detects; -EDEADLK.
- [ ] AC-10: pthread_mutex priority inheritance: high-prio thread doesn't priority-invert.
- [ ] AC-11: Robust-futex on owner-exit: FUTEX_OWNER_DIED set in word.

## Architecture

```
struct PiState {
  list: ListHead,                       // task.pi_state_list
  pi_mutex: RtMutex,
  refcount: AtomicUsize,
  owner: Option<*TaskStruct>,
  key: FutexKey,
  uaddr: *mut u32,
}

struct FutexQ {
  task: *TaskStruct,
  pi_state: Option<RcuPtr<PiState>>,
  /* ... base fields */
}
```

`Futex::lock_pi(uaddr, flags, timeout) -> Result<()>`:
1. /* Look up futex_key */
2. ret = get_futex_key(uaddr, flags & FLAGS_SHARED, &key, FUTEX_WRITE).
3. if ret: return ret.
4. /* Hash bucket lock */
5. hb = hash_futex(&key); spin_lock(&hb.lock).
6. /* Atomic try */
7. q.task = current; q.pi_state = None.
8. ret = Futex::lock_pi_atomic(uaddr, hb, &key, &q.pi_state, &exiting).
9. if ret == 1:
   - /* Fast-path acquired */
   - spin_unlock(&hb.lock).
   - return Ok(()).
10. if ret < 0:
    - spin_unlock(&hb.lock).
    - return Err(ret).
11. /* Slow-path: queue + rt_mutex_timed_futex_lock */
12. __queue_me(&q, hb).
13. spin_unlock(&hb.lock).
14. ret = rt_mutex_timed_futex_lock(&q.pi_state.pi_mutex, timeout).
15. /* Re-acquire hb-lock */
16. spin_lock(&hb.lock).
17. ret_fixup = Futex::fixup_pi_state_owner(uaddr, q.pi_state, current, ret).
18. unqueue_me_pi(&q).
19. spin_unlock(&hb.lock).
20. return ret.

`Futex::lock_pi_atomic(uaddr, hb, key, ps, exiting)`:
1. /* Read uval atomically */
2. ret = get_futex_value_locked(&uval, uaddr).
3. if ret: return ret.
4. ownertid = uval & FUTEX_TID_MASK.
5. if ownertid == 0:
   - /* Try fast acquire */
   - new = current.pid.
   - ret = futex_atomic_cmpxchg_inatomic(uval, uaddr, 0, new).
   - if ret == 0: return 1 (acquired).
   - if ret == -EFAULT: return -EFAULT.
6. /* Owner exists; possibly stale */
7. if uval & FUTEX_OWNER_DIED:
   - return Futex::handle_owner_died(uaddr, key, exiting).
8. /* Attach pi_state */
9. ret = Futex::lookup_pi_state(uaddr, uval, hb, key, ps, exiting).
10. /* Set FUTEX_WAITERS bit */
11. ret = futex_atomic_cmpxchg_inatomic(uval, uaddr, uval, uval | FUTEX_WAITERS).
12. return 0 (queued).

`Futex::unlock_pi(uaddr, flags) -> Result<()>`:
1. ret = get_futex_key(uaddr, flags, &key, FUTEX_WRITE).
2. if ret: return ret.
3. hb = hash_futex(&key); spin_lock(&hb.lock).
4. /* Read uval */
5. ret = get_futex_value_locked(&uval, uaddr).
6. if ret: goto out.
7. vpid = task_pid_vnr(current).
8. if (uval & FUTEX_TID_MASK) != vpid: ret = -EPERM; goto out.
9. /* Find pi_state via top-waiter */
10. top_waiter = futex_top_waiter(hb, &key).
11. if !top_waiter:
    - /* No waiters: clear */
    - cmpxchg uaddr (uval) → 0.
12. else:
    - new_owner = top_waiter.task.
    - new_uval = task_pid_vnr(new_owner) | (multi_waiters ? FUTEX_WAITERS : 0).
    - cmpxchg uaddr (uval) → new_uval.
    - /* Wake */
    - Futex::wake_pi(top_waiter).
13. out:
14. spin_unlock(&hb.lock).
15. return ret.

`Futex::wake_pi(top_waiter)`:
1. /* rt_mutex grants ownership transfer */
2. rt_mutex_futex_unlock(top_waiter.pi_state.pi_mutex).
3. /* Wake task */
4. wake_up_state(top_waiter.task, TASK_NORMAL).

`Futex::handle_owner_died(uaddr, key, exiting)`:
1. /* OWNER_DIED bit set; we may inherit */
2. if uval & FUTEX_TID_MASK == 0:
   - /* Owner dead and unlocked */
   - cmpxchg uaddr (uval) → current.pid.
   - return 1 (acquired).
3. /* Owner dead but task still in some pi-list (race) */
4. exiting.task = lookup_owner; exiting.in_progress = true.
5. return -EAGAIN.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pi_state_refcount_balanced` | INVARIANT | per-pi_state alloc/get/put balanced. |
| `owner_tid_consistent_with_uaddr` | INVARIANT | per-pi_state.owner.pid == uaddr & FUTEX_TID_MASK ∨ OWNER_DIED. |
| `waiters_bit_set_when_pi_state_has_waiters` | INVARIANT | per-pi_state with rt_mutex waiters: FUTEX_WAITERS in uaddr. |
| `top_waiter_task_alive` | INVARIANT | per-wake_pi: top_waiter.task ref held. |
| `task_pi_state_list_inc` | INVARIANT | per-attach: pi_state on owner.pi_state_list. |
| `pi_state_freed_after_owner_release` | INVARIANT | per-unlock_pi: pi_state refcount-- to 0 ⟹ free. |

### Layer 2: TLA+

`kernel/futex/pi.tla`:
- Per-LOCK_PI / UNLOCK_PI / TRYLOCK_PI + per-rt_mutex chain.
- Per-priority-inheritance + per-deadlock-detection.
- Properties:
  - `safety_no_priority_inversion` — per-PI: high-prio waiter ⟹ owner.prio ≥ waiter.prio.
  - `safety_at_most_one_owner` — per-pi_state: at most one owner.
  - `safety_owner_died_eventually_handled` — per-OWNER_DIED: cleared on next-acquire.
  - `liveness_per_unlock_eventually_wakes_top` — per-unlock_pi: top-waiter wake.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Futex::lock_pi` post: acquired (uaddr & TID == current.pid) ∨ blocked-and-fixup | `Futex::lock_pi` |
| `Futex::lock_pi_atomic` post: ret ∈ {1, 0, -EFAULT, -ESRCH} | `Futex::lock_pi_atomic` |
| `Futex::unlock_pi` post: uaddr ∈ {0, top_waiter.tid | WAITERS-if-more} | `Futex::unlock_pi` |
| `Futex::handle_owner_died` post: cleanup applied or -EAGAIN | `Futex::handle_owner_died` |
| `Futex::wake_pi` post: rt_mutex passed; task woken | `Futex::wake_pi` |

### Layer 4: Verus/Creusot functional

`Per-FUTEX_LOCK_PI: cmpxchg-fast or rt_mutex-slow with PI-propagation; per-FUTEX_UNLOCK_PI: clear-or-pass to top-waiter; per-FUTEX_OWNER_DIED handled` semantic equivalence: per-Documentation/locking/pi-futex.rst + per-rt_mutex semantics.

## Hardening

(Inherits row-1 features from `kernel/futex/core.md` § Hardening.)

PI-futex reinforcement:

- **Per-uval cmpxchg-strict (avoid stale)** — defense against per-ABA.
- **Per-pi_state refcount strict** — defense against per-pi_state UAF.
- **Per-rt_mutex chain-walk deadlock detection** — defense against per-cycle hang.
- **Per-OWNER_DIED handled by next acquirer** — defense against per-stuck-mutex.
- **Per-task PF_EXITING handled (handle_exit_race)** — defense against per-attach-to-exiting.
- **Per-FUTEX_TID_MASK strict format** — defense against per-format-spoof.
- **Per-FUTEX_WAITERS bit cmpxchg-set** — defense against per-missing-wake.
- **Per-priority-boost rt_mutex backed (no manual)** — defense against per-priority-inversion.
- **Per-robust-futex exit cleanup** — defense against per-stale-mutex-after-task-exit.
- **Per-CAP_SYS_NICE for SCHED_FIFO via PI** — defense against unprivileged real-time.
- **Per-pi_state.owner == NULL after final put** — defense against per-stale-owner deref.
- **Per-fixup_pi_state_owner post-rt_mutex** — defense against per-mid-rt-mutex preemption inconsistency.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — copy-in of futex words and robust-list head pointers is bounds-checked; user-supplied robust_list_head cannot pivot reads into adjacent VMAs.
- **PAX_KERNEXEC** — `rt_mutex` and pi-state callback tables (`waiter_clb`, `rt_mutex_setprio`) are W^X.
- **PAX_RANDKSTACK** — randomize kernel stack on `futex_lock_pi`, `futex_unlock_pi`, and `exit_robust_list` entry.
- **PAX_REFCOUNT** — saturating atomics on `futex_pi_state.refcount` and per-hb plist refs; underflow on `put_pi_state` traps.
- **PAX_MEMORY_SANITIZE** — scrub freed `futex_pi_state` on RCU release so a re-allocated slab object cannot leak old `owner`/`rt_waiter`.
- **PAX_UDEREF** — strict user/kernel split when chasing `robust_list_head.list_op_pending`; the kernel never dereferences a user pointer as a kernel one.
- **PAX_RAP / kCFI** — type-signed indirect calls for the rt_mutex backend (`->prepare_lock`, `->wake_q_add`, `task_blocks_on_rt_mutex`).
- **GRKERNSEC_HIDESYM** — hide `pi_state_cache`, `futex_q`, `rt_mutex_*` symbols from non-root /proc/kallsyms.
- **GRKERNSEC_DMESG** — restrict pi-state inconsistency warnings ("pi futex: ... ") to CAP_SYSLOG.
- **rt_mutex backend PAX_RAP** — the PI boost path goes through type-signed indirect calls so an attacker who plants a fake `rt_waiter` (e.g. via heap UAF) cannot redirect `rt_mutex_top_waiter` chasing.
- **FUTEX_OWNER_DIED handling** — when the kernel observes OWNER_DIED, it must run `fixup_pi_state_owner` under hb->lock with the pi_state ref held; the bit cannot be set by user-space writes after exit cleanup.
- **Robust-list cleanup on exit** — `exit_robust_list` walks the user-supplied list with a bounded iteration limit and PAX_USERCOPY checks per entry; a circular or oversized list cannot loop the exiting task or read out-of-VMA memory.
- **TID-mask integrity** — `FUTEX_TID_MASK` is enforced when matching user-supplied owner tids; bits outside the mask are rejected, denying format-spoof tricks that smuggle WAITERS or OWNER_DIED into the tid.
- **CAP_SYS_NICE for SCHED_FIFO via PI** — a PI boost that would lift the holder to a real-time class requires the original owner already hold CAP_SYS_NICE, denying unprivileged real-time elevation through PI inheritance.
- **No PI on shared anon without VM_LOCKED check** — PI futexes on pages that can be swapped out are explicitly rejected to avoid the well-known page-migration vs pi_state race class.
- **Rationale**: PI futexes splice user-controlled state (the futex word, robust list, TID) into the rt_mutex inheritance graph. PAX_RAP on the rt_mutex backend plus strict TID-mask and robust-list bounding turn a historically CVE-rich area (CVE-2014-3153 class) into a typed, capability-gated path.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/futex/core.c (covered in `core.md` Tier-3)
- kernel/futex/waitwake.c (covered separately if expanded)
- kernel/futex/requeue.c CMP_REQUEUE_PI (covered separately if expanded)
- kernel/locking/rtmutex.c (covered separately if expanded)
- kernel/exit.c exit_robust_list (covered separately if expanded)
- Implementation code
