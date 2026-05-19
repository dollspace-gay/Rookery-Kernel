# Tier-3: kernel/futex/core.c — futex (fast userspace mutex) per-key hash + waitqueue + atomic-get

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/futex/00-overview.md
upstream-paths:
  - kernel/futex/core.c
  - kernel/futex/futex.h
  - kernel/futex/syscalls.c
  - kernel/futex/waitwake.c
  - kernel/futex/pi.c
  - kernel/futex/requeue.c
-->

## Summary

Futex (Fast Userspace Mutex) is the kernel primitive that backs userspace pthread_mutex / pthread_cond / std::sync::Mutex / Rust Tokio etc. — userspace fast-path is purely user-mode CAS on a 32-bit value; kernel only involved on contention via `futex(2)` syscall (FUTEX_WAIT / FUTEX_WAKE / FUTEX_REQUEUE / FUTEX_LOCK_PI / FUTEX_UNLOCK_PI / FUTEX_WAIT_BITSET). core.c manages the per-(mm, addr) keys + per-key hash bucket + per-bucket waitqueue. waitwake.c is the FUTEX_WAIT/FUTEX_WAKE op core; pi.c is FUTEX_LOCK_PI (priority-inheritance via rt_mutex backend); requeue.c is FUTEX_CMP_REQUEUE for cond-var broadcasts.

This Tier-3 covers `core.c` (~2015) + `futex.h` (~461) + `waitwake.c` (~755) + `requeue.c` (~918) + `pi.c` (~1304) + `syscalls.c`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `union futex_key` | per-(mm, addr) key | `kernel::futex::FutexKey` |
| `struct futex_q` | per-waiter queue entry | `FutexQ` |
| `struct futex_hash_bucket` | per-bucket waitqueue+lock | `FutexHashBucket` |
| `struct futex_pi_state` | per-PI-mutex state | `FutexPiState` |
| `get_futex_key(uaddr, fshared, key, rw)` | per-(addr, mm/inode) key derive | `Futex::get_key` |
| `get_futex_value_locked(uval, uaddr)` | atomic uaddr fetch | `Futex::get_value_locked` |
| `futex_hash(key)` | per-key bucket lookup | `Futex::hash` |
| `futex_q_lock(q)` / `_unlock(q)` | per-bucket lock | `FutexHashBucket::lock` / `_unlock` |
| `futex_wait(uaddr, flags, val, abs_time, bitset)` | FUTEX_WAIT impl | `Futex::wait` |
| `futex_wake(uaddr, flags, nr_wake, bitset)` | FUTEX_WAKE impl | `Futex::wake` |
| `futex_requeue(uaddr1, flags1, uaddr2, flags2, nr_wake, nr_requeue, cmpval, requeue_pi)` | FUTEX_REQUEUE | `Futex::requeue` |
| `futex_lock_pi(uaddr, flags, time, trylock)` (pi.c) | FUTEX_LOCK_PI | `Futex::lock_pi` |
| `futex_unlock_pi(uaddr, flags)` | FUTEX_UNLOCK_PI | `Futex::unlock_pi` |
| `futex_wait_setup(uaddr, ...)` | wait-prep (verify uval == val) | `Futex::wait_setup` |
| `futex_wait_queue_me(...)` | enqueue + sleep | `Futex::wait_queue_me` |
| `mark_wake_futex(...)` | wake helper | `Futex::mark_wake_futex` |
| `lookup_pi_state(...)` (pi.c) | per-key PI state lookup | `FutexPi::lookup_state` |
| `attach_to_pi_owner(...)` / `attach_to_pi_state(...)` | PI ownership chain mgmt | `FutexPi::attach_to_owner` |
| `requeue_pi_wake_futex(...)` | PI-aware requeue wake | `FutexPi::requeue_pi_wake` |
| `__futex_atomic_op_inuser(...)` (arch) | per-arch atomic-op | `arch::futex::atomic_op` |
| `cmpxchg_futex_value_locked(...)` | per-arch atomic-CAS | `arch::futex::cmpxchg_value_locked` |

## Compatibility contract

REQ-1: `futex_key` discriminates between:
- Shared futex (in shared file mapping; key = (inode, page-offset, page-bits)).
- Private futex (in process private mapping; key = (mm, uaddr-aligned, hash-bits)).

REQ-2: Per-(mm, uaddr) hashbucket via Jenkins hash → bucket index modulo `futex_hashsize` (sized as 256 × num_possible_cpus, capped).

REQ-3: Per-bucket `futex_hash_bucket`:
- `lock` (raw spinlock).
- `chain` (plist_head; sorted by waiter priority).
- `waiters` (atomic counter).

REQ-4: Per-waiter `futex_q`:
- `list` (plist_node).
- `task` (waiter task pointer).
- `lock_ptr` (back-ref to bucket lock).
- `key` (futex_key).
- `bitset` (FUTEX_WAIT_BITSET bitmask).
- `pi_state` (Option for PI futexes).
- `rt_waiter` (rt_mutex waiter for PI).
- `requeue_pi_key` (target key for FUTEX_REQUEUE_PI).

REQ-5: FUTEX_WAIT semantics:
1. get_futex_key(uaddr) → key.
2. queue_lock(hb).
3. uval := get_futex_value_locked(uaddr).
4. If uval != val: queue_unlock(hb); return -EWOULDBLOCK.
5. Insert futex_q into hb.chain.
6. Set task state TASK_INTERRUPTIBLE.
7. queue_unlock(hb).
8. Schedule (sleep).
9. On wakeup: remove from chain (if not already removed by waker).
10. Return 0 (woken) / -ETIMEDOUT / -ERESTARTSYS / -EWOULDBLOCK.

REQ-6: FUTEX_WAKE semantics:
1. get_futex_key(uaddr) → key.
2. queue_lock(hb).
3. plist_for_each_entry_safe(q, tmp, &hb.chain, list):
   - If q.key matches AND (q.bitset & bitset) != 0:
     - mark_wake_futex(q): q.lock_ptr = NULL; wake_q_add(&wake_q, q.task); plist_del(&q.list).
     - count++.
     - If count == nr_wake: break.
4. queue_unlock(hb).
5. wake_up_q(&wake_q) — actual wakeup outside lock.

REQ-7: FUTEX_REQUEUE:
1. Two-key acquire: hb1, hb2 (per-bucket lock; ordered to avoid deadlock).
2. Optional cmpxchg-cmpval (FUTEX_CMP_REQUEUE).
3. Wake nr_wake from hb1.
4. Move nr_requeue from hb1.chain to hb2.chain (key updated to uaddr2's key).
5. Used by glibc pthread_cond_broadcast: wake one + requeue rest from cond-wait queue to mutex queue.

REQ-8: FUTEX_LOCK_PI semantics (pi.c):
1. attach_to_pi_owner: lookup futex_pi_state for this key; if none, create + boost owner via rt_mutex.
2. rt_mutex_timed_futex_lock — task blocks via rt_mutex (PI propagation).
3. On acquire: futex value updated to (TID | FUTEX_WAITERS-bit-cleared).
4. Return 0.

REQ-9: FUTEX_UNLOCK_PI:
1. lookup_pi_state.
2. wake highest-priority waiter via rt_mutex_futex_unlock.
3. Update uaddr to new owner-TID via cmpxchg.

REQ-10: Robust-list integration:
- per-task `robust_list_head` registered via set_robust_list(2).
- On task exit: kernel walks list to clean-up futexes the dying task held; sets FUTEX_OWNER_DIED bit + wakes one waiter.

REQ-11: Pi-aware requeue (FUTEX_REQUEUE_PI):
- Used by glibc pthread_cond_broadcast for PI-mutex wait+wake.
- Combines FUTEX_WAIT + lazy FUTEX_LOCK_PI on requeue target.

REQ-12: futex(2) syscall ops:
- FUTEX_WAIT(0), _WAKE(1), _FD(2), _REQUEUE(3), _CMP_REQUEUE(4), _WAKE_OP(5), _LOCK_PI(6), _UNLOCK_PI(7), _TRYLOCK_PI(8), _WAIT_BITSET(9), _WAKE_BITSET(10), _WAIT_REQUEUE_PI(11), _CMP_REQUEUE_PI(12), _LOCK_PI2(13).

REQ-13: arch_atomic_op_in_user_mode for FUTEX_WAKE_OP:
- Atomic-op (ADD/AND/OR/XOR/SET) on user uaddr2; compare result + wake hb1 + hb2 conditionally.
- Used by libc for "decrement-and-wake" idioms.

## Acceptance Criteria

- [ ] AC-1: pthread_mutex_lock/unlock with single waiter: glibc test passes.
- [ ] AC-2: pthread_cond_signal / pthread_cond_broadcast: 100 threads waiting; broadcast wakes all.
- [ ] AC-3: PI mutex: 3-thread chain (low → med → high) with low holding mutex; high waits → low boost-priority observed via /proc/N/sched.
- [ ] AC-4: FUTEX_WAKE_OP: glibc semaphore_post benchmark passes.
- [ ] AC-5: Robust-list cleanup: process holding futex SIGKILLs; FUTEX_OWNER_DIED bit set; next waiter gets EOWNERDEAD.
- [ ] AC-6: futex stress: glibc nptl test suite full pass on 32-CPU host.
- [ ] AC-7: Multi-process shared futex: 2 procs share IPC seg; futex word in shared memory; cross-process WAIT/WAKE works.
- [ ] AC-8: futex hash collision: 1M private futexes per process; hash bucket distribution measured; no pathological clustering.

## Architecture

`Futex` global registry:

```
struct Futex {
  hash: KBox<[FutexHashBucket; FUTEX_HASHSIZE]>,
}

struct FutexHashBucket {
  lock: RawSpinLock<()>,
  chain: PListHead,                          // priority-sorted plist
  waiters: AtomicI32,
}

union FutexKey {
  shared: KeyShared,                          // (inode, page_off, hash_bits)
  private: KeyPrivate,                        // (mm, uaddr_page, hash_bits)
  both: KeyBoth,                              // for hashing; ptr+word+offset
}

struct FutexQ {
  list: PListNode,
  task: KArc<TaskStruct>,
  lock_ptr: *const RawSpinLock<()>,           // back-ref to bucket lock
  key: FutexKey,
  bitset: u32,
  pi_state: Option<KArc<FutexPiState>>,
  rt_waiter: Option<RtMutexWaiter>,
  requeue_pi_key: Option<FutexKey>,
}

struct FutexPiState {
  list: ListNode,                              // global pi-state list
  rtmutex: RtMutex,                            // PI-protected mutex
  refcount: AtomicI32,
  key: FutexKey,
  owner: KWeak<TaskStruct>,
}
```

`Futex::wait(uaddr, flags, val, abs_time, bitset)` flow:
1. key := get_futex_key(uaddr, flags & FUTEX_PRIVATE).
2. hb := futex_hash(key).
3. queue_lock(hb).
4. uval := get_futex_value_locked(uaddr).
5. If uval != val: queue_unlock; return -EAGAIN.
6. q := init_futex_q(key, bitset, current_task).
7. plist_add(&q.list, &hb.chain).
8. queue_unlock.
9. set_current_state(TASK_INTERRUPTIBLE).
10. If abs_time: schedule_hrtimeout(abs_time, HRTIMER_MODE_ABS).
    Else: schedule().
11. set_current_state(TASK_RUNNING).
12. queue_lock(hb).
13. If still in chain: plist_del; ret = -ETIMEDOUT or -ERESTARTSYS.
14. queue_unlock.
15. Return ret.

`Futex::wake(uaddr, flags, nr_wake, bitset)`:
1. key := get_futex_key.
2. hb := futex_hash.
3. queue_lock(hb).
4. wake_q := WAKE_Q_HEAD_INIT.
5. plist_for_each_entry_safe(q, tmp, &hb.chain):
   - If q.key == key AND (q.bitset & bitset):
     - mark_wake_futex(q, &wake_q).
     - count++.
     - If count == nr_wake: break.
6. queue_unlock(hb).
7. wake_up_q(&wake_q).
8. Return count.

`Futex::lock_pi(uaddr, flags, time, trylock)` (pi.c):
1. key := get_futex_key.
2. hb := futex_hash.
3. queue_lock.
4. pi_state := lookup_pi_state(uaddr, key, hb).
5. If !pi_state: futex_lock_pi_atomic — atomic CAS uaddr to current TID; if success, return 0.
6. attach_to_pi_state(current, pi_state); insert q into hb.chain.
7. queue_unlock.
8. rt_mutex_timed_futex_lock(&pi_state.rtmutex, time) → blocks with PI boost.
9. On acquire: cleanup q.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `futex_q_no_uaf` | UAF | per-FutexQ on stack of waiter; lock_ptr cleared by waker before walker exits, but waker holds bucket-lock while walker may read. |
| `key_uniqueness` | INVARIANT | (mm, uaddr) → unique private key; (inode, offset) → unique shared key. |
| `bucket_chain_priority_sorted` | INVARIANT | per-bucket plist sorted by waiter priority (defense against unfair-wake). |
| `pi_state_refcount_no_underflow` | INVARIANT | FutexPiState.refcount ≥ 0; defense against double-decrement on requeue races. |
| `cmpxchg_user_safe` | INVARIANT | get_futex_value_locked uses copy_from_user that handles page-fault; defense against guest unmap mid-read causing kernel oops. |

### Layer 2: TLA+

`kernel/futex/wait_wake.tla`:
- Per-waiter state ∈ {Awake, Queued, Sleeping, WakeRequested, Done}.
- Transitions per WAIT/WAKE/timeout/signal.
- Properties:
  - `safety_value_check_atomic` — WAIT-EAGAIN if uval != val (read under bucket-lock).
  - `safety_no_lost_wake` — WAKE wakes ≤ nr_wake matching waiters; never wakes non-matching.
  - `liveness_woken_eventually_runs` — WakeRequested → Done unless preempted indefinitely (assumed bounded).

`kernel/futex/pi_chain.tla` (refines from rt_mutex semantics):
- PI-state state ∈ {NoOwner, Owned(task), OwnerExited, BeingDeleted}.
- Properties:
  - `safety_pi_owner_unique` — at most one Owned-task per PI-state at a time.
  - `safety_owner_died_propagates` — OwnerExited transitions waiter to EOWNERDEAD path.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Futex::wait` post: q removed from chain; wake-status reflects wake/timeout/signal | `Futex::wait` |
| `Futex::wake` post: woken count ≤ nr_wake; per-q.lock_ptr == NULL after mark_wake_futex | `Futex::wake` |
| `Futex::requeue` post: nr_wake woken from src; nr_requeue moved to dst-bucket | `Futex::requeue` |
| `FutexPi::lookup_state` post: refcount incremented if state found; defense against use-after-decref | `FutexPi::lookup_state` |
| Per-bucket plist sorted invariant maintained across add/del | `Futex::wait` insert |

### Layer 4: Verus/Creusot functional

`pthread_cond_broadcast = futex_wake(cond, INT_MAX) → all waiters in cond's bucket eventually woken` semantic equivalence: every queued waiter at broadcast time eventually transitions to Done state.

## Hardening

(Inherits row-1 features from `kernel/futex/00-overview.md` § Hardening.)

futex-specific reinforcement:

- **Per-bucket lock fairness via plist** — defense against priority-inversion on contended futex.
- **Robust-list per-task cleanup at exit** — defense against process-crash leaving futex stuck in OWNER_DIED state.
- **PI rt_mutex backend** — defense against priority inversion in priority-inheriting futex.
- **Per-key derivation under page-pinning** — defense against page-migration changing key mid-WAIT.
- **cmpxchg_futex_value_locked under bucket-lock** — defense against TOCTOU between value-check + wait-enqueue.
- **bitset must be nonzero** for WAIT_BITSET — defense against ambiguous "wake all" intent.
- **futex_hashsize bounded** — defense against attacker creating many futexes to overflow per-bucket chain.
- **PI-state refcount** — defense against use-after-free via concurrent requeue.
- **Per-task max-pi-state count** capped — defense against runaway PI-chain growth.
- **set_robust_list per-task validates list addr** — defense against attacker-controlled robust-list pointing to kernel memory.
- **futex(2) flags FUTEX_PRIVATE_FLAG honored** — defense against shared-futex-key-leak across mm.
- **wake_up_q outside bucket-lock** — defense against waiter scheduling-in racing with waker exit; reduces lock-hold time.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on task/cred/sched_attr buffers; rejects oversized robust_list_head copies and abs_time timespec copies.
- **PAX_KERNEXEC** — W^X for BPF JIT'd code, kprobe/uprobe trampolines that may fire under bucket-lock.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization; obscures `FutexQ`-on-stack layout against targeted overwrite by a racing exit handler.
- **PAX_REFCOUNT** — saturating refcount on task_struct (waiter), pi_state, cred, mm_struct, files_struct.
- **PAX_MEMORY_SANITIZE** — zero-on-free for FutexPiState, task_struct; prevents stale pi-state aliasing after rt_mutex release.
- **PAX_MEMORY_STACKLEAK** — kernel-stack zeroing on syscall exit; wipes leaked `FutexQ` and `FutexKey` stack frames between futex(2) calls.
- **PAX_UDEREF** — strict user-pointer access for cmpxchg_futex_value_locked and copy_from_user on robust_list_head; refuses kernel-pointer uaddr.
- **PAX_RAP / kCFI** — indirect-call signature enforcement for rt_mutex wake-callback and futex_q->lock_ptr back-reference dispatch.
- **GRKERNSEC_HIDESYM** — hide kernel addresses in /proc/<pid>/* + kallsyms; suppresses futex_q / pi_state pointer disclosure via /proc/<pid>/wchan.
- **GRKERNSEC_HARDEN_PTRACE** — restrict ptrace cross-uid (Yama scope ≥ 1); blocks attacker from inspecting a victim's futex robust_list registration.
- **GRKERNSEC_BRUTE** — exponential delay on consecutive brute attempts; throttles bucket-collision flooding probes.
- **GRKERNSEC_HARDEN_IPC** — restrict cross-uid FUTEX_WAKE on shared-mapping futexes to same-owner mappings.
- **GRKERNSEC_KSTACKOVERFLOW** — kernel-stack overflow guard against deep PI-chain boost walks.
- **GRKERNSEC_DMESG** — restrict syslog so robust-list cleanup WARN/oops traces leaking pi_state pointers are unreadable to unprivileged users.
- **GRKERNSEC_SYSCTL_DISABLE** — disable dangerous sysctls (kernel.futex_hashsize tunable) by default-locked.
- **GRKERNSEC_CONFIG_AUDIT** — boot-time runtime-config integrity check; verifies FUTEX/FUTEX_PI/ROBUST_LIST profile matches signed config.
- **Per-mm-uaddr key salt** — defense against cross-process futex-key prediction; salt seeded at boot from RDRAND.

Per-doc rationale: futex backs every userspace lock in the system, so a corrupted `pi_state` or stale `FutexQ` pointer is a direct kernel-level UAF/EoP primitive; the robust-list cleanup path operates on attacker-controllable userspace structures during exit. PaX UDEREF + USERCOPY prevent kernel-pointer poisoning of uaddr, REFCOUNT/MEMORY_SANITIZE close the pi_state UAF window across requeue races, RAP/kCFI hardens the rt_mutex wake-callback dispatch, and Grsecurity HIDESYM/HARDEN_PTRACE/HARDEN_IPC eliminate the cross-process futex-key disclosure channels that vanilla Linux exposes through /proc/<pid>/wchan.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- rt_mutex (covered in `kernel/locking/rtmutex.md` future Tier-3)
- pthread library glue (userspace concern)
- compat-32 syscall layer (covered separately if needed)
- Implementation code
