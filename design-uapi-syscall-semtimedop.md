---
title: "Tier-5 syscall: semtimedop(2) — syscall 220"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`semtimedop(2)` is the timeout-bounded variant of `semop(2)`. It performs an atomic vector of System V semaphore operations on `semid`. If the vector cannot proceed immediately and `IPC_NOWAIT` is not set, the caller blocks for at most `*timeout` (a relative `CLOCK_MONOTONIC`-style timespec), returning `-EAGAIN` on timer expiry.

A `NULL` timeout argument is equivalent to `semop(2)` (block indefinitely). The timeout is **relative**, **monotonic**, and at nanosecond resolution. `semtimedop` is the preferred entry point on modern architectures; some ports (arm64, riscv) expose only `semtimedop` and route `semop(2)` to it. Critical for: latency-bounded synchronization (real-time daemons, transactional engines).

### Acceptance Criteria

- [ ] AC-1: Timeout = 100ms; sem blocked, no signaller: returns `-EAGAIN` after >= 100ms.
- [ ] AC-2: Timeout = 100ms; signaller posts before 100ms: returns 0 quickly.
- [ ] AC-3: Timeout `(0, 0)` and blocking op: returns `-EAGAIN` immediately.
- [ ] AC-4: `timeout = NULL`: blocks indefinitely (same as semop).
- [ ] AC-5: `tv_nsec = 1e9`: returns `-EINVAL`.
- [ ] AC-6: `tv_sec = -1`: returns `-EINVAL`.
- [ ] AC-7: Signal during wait: returns `-EINTR` (not `-EAGAIN`).
- [ ] AC-8: RMID during wait: returns `-EIDRM`.
- [ ] AC-9: Vector applies atomically on wake-before-timeout.
- [ ] AC-10: `nsops > SEMOPM`: `-E2BIG`.
- [ ] AC-11: All other `semop` ACs inherited.

### Architecture

```rust
#[syscall(nr = 220, abi = "sysv")]
pub fn sys_semtimedop(
    semid: i32, sops: UserPtr<Sembuf>, nsops: usize,
    timeout: UserPtr<KernelTimespec>,
) -> isize {
    let to = if timeout.is_null() {
        None
    } else {
        let mut ts = KernelTimespec::default();
        match unsafe { timeout.copy_in(&mut ts) } {
            Ok(()) => {
                if ts.tv_sec < 0 || ts.tv_nsec < 0 || ts.tv_nsec >= 1_000_000_000 {
                    return -EINVAL;
                }
                Some(ts.to_ktime())
            }
            Err(_) => return -EFAULT,
        }
    };
    Ipc::do_semtimedop(semid, sops, nsops, to)
}
```

`Ipc::do_semtimedop(semid, uptr, nsops, timeout) -> isize`:
1. (see `semop.md` REQ-1..REQ-12 — same path)
2. /* Block path */
3. if cannot complete and !IPC_NOWAIT {
4.   match timeout {
5.     None => sleep_on_queue_indefinite()?,
6.     Some(t) if t.is_zero() => return -EAGAIN,
7.     Some(t) => {
8.       let exp = ktime_add(ktime_get(), t);
9.       hrtimer_arm(exp, |q| wake_queue_eagain(q));
10.      sleep_on_queue_until_wake_or_timeout()?;
11.    }
12.  }
13. }

`Ipc::sleep_on_queue_until_wake_or_timeout() -> Result<(), Errno>`:
1. enqueue_self_on_pending(sma, sops);
2. unlock_sma(sma);
3. let r = schedule_timeout_interruptible(remaining);
4. lock_sma(sma);
5. /* On wake: */
6. if signal_pending() { dequeue_self(); return Err(EINTR); }
7. if sma.sem_perm.deleted { dequeue_self(); return Err(EIDRM); }
8. if r == 0 /* timer expired */ {
9.   dequeue_self();
10.  return Err(EAGAIN);
11. }
12. /* else: signalled by another caller's update_queue; re-test by caller */
13. Ok(())

### Out of Scope

- Non-timed semop (covered in `semop.md`).
- Set admin (covered in `semctl.md`).
- Allocation (covered in `semget.md`).
- 32-bit `semtimedop_time64` (covered separately under compat-layer).
- Implementation code.

### signature

```c
int semtimedop(int semid, struct sembuf *sops, size_t nsops,
               const struct timespec *timeout);
```

```c
struct timespec {
    __kernel_time_t tv_sec;
    long            tv_nsec;
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `semid`   | `int` | in | Set id from `semget(2)`. |
| `sops`    | `struct sembuf *` | in | Vector of operations. |
| `nsops`   | `size_t` | in | Vector length, `1..=SEMOPM`. |
| `timeout` | `const struct timespec *` | in | Relative timeout; `NULL` ⟹ block indefinitely. |

### return value

| Value | Meaning |
|---|---|
| `0` | All ops applied. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `E2BIG` | `nsops > SEMOPM`. |
| `EACCES` | Insufficient permission. |
| `EAGAIN` | `IPC_NOWAIT` set and would block, or timeout expired. |
| `EFAULT` | `sops` or `timeout` not addressable. |
| `EFBIG` | `sem_num >= sma->sem_nsems`. |
| `EIDRM` | Set removed while blocked. |
| `EINTR` | Signal caught while blocked. |
| `EINVAL` | Bad `semid`, `nsops == 0`, or `tv_nsec < 0` / `tv_nsec >= 1e9` / `tv_sec < 0`. |
| `ENOMEM` | Out of memory for SEM_UNDO. |
| `ERANGE` | semval > SEMVMX or undo > SEMAEM. |

### abi surface

```text
__NR_semtimedop  (x86_64)        = 220
__NR_semtimedop  (arm64)         = 192
__NR_semtimedop  (riscv)         = 192 (semtimedop_time64)
__NR_semtimedop_time64 (i386 32) = 412 (y2038 ABI)

Time clock: CLOCK_MONOTONIC equivalent (kernel uses ktime_get).
Relative timeout (NOT absolute).
NULL timeout ⟹ infinite block (same as semop(2)).
```

### compatibility contract

REQ-1: Syscall number is **220** on x86_64. ABI-stable.

REQ-2: `timeout == NULL`: identical to `semop(2)`; block indefinitely.

REQ-3: `timeout != NULL`: copy_from_user → `struct timespec`. Validate:
- `tv_sec >= 0`,
- `0 <= tv_nsec < 1_000_000_000`.
Otherwise `EINVAL`.

REQ-4: Timeout interpretation: **relative** duration measured against monotonic clock. `(0,0)` ⟹ equivalent to `IPC_NOWAIT` semantics (do not block; return EAGAIN if vector cannot complete).

REQ-5: All other semantics inherited from `semop(2)` REQ-3..REQ-12 (vector atomicity, SEM_UNDO accounting, permission ladder, LSM hook, etc.).

REQ-6: 32-bit ABI `semtimedop_time64` (syscall 412 on i386) uses `struct __kernel_timespec` (64-bit `tv_sec`) for y2038 safety.

REQ-7: Wake on timer fire path: `hrtimer_setup` armed; wake routine pulls caller from `sma->pending_alter` / `pending_const`, sets return to `-EAGAIN`.

REQ-8: Signal vs timer race: signal preempts; return is `-EINTR` (kernel never converts `EINTR` to `EAGAIN`).

REQ-9: Remaining time on EINTR/EIDRM not returned to user (semtimedop does NOT have a `remain` out-parameter like nanosleep). This is intentional and POSIX-conformant.

REQ-10: If the vector can complete immediately on entry, no timer is armed (fast path); the timeout argument is consulted but never tested.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `timeout_validation` | INVARIANT | tv_sec<0 ∨ tv_nsec∉[0,1e9) ⟹ EINVAL. |
| `null_timeout_block_forever` | INVARIANT | timeout=NULL behaves as semop. |
| `zero_timeout_eagain` | INVARIANT | timeout=(0,0) on blocking op ⟹ EAGAIN immediate. |
| `intr_beats_timer` | INVARIANT | signal during wait ⟹ EINTR, not EAGAIN. |
| `eidrm_beats_timer` | INVARIANT | RMID during wait ⟹ EIDRM. |
| `no_remain_returned` | INVARIANT | semtimedop does not write remaining to user. |

### Layer 2: TLA+

`ipc/semtimedop.tla`:
- States: copy-timeout, validate, arm-timer, sleep, wake (timer / signal / rmid / success).
- Properties:
  - `safety_timer_strict` — wake-due-to-timer ⟹ EAGAIN; never silent success.
  - `safety_signal_priority` — signal pre-empts timer.
  - `safety_atomic_apply_on_success` — vector atomically applied at wake-by-update_queue.
  - `liveness_timer_fires` — armed timer eventually wakes or is cancelled.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_semtimedop` post: timer-fire ⟹ vector unchanged | `Ipc::do_semtimedop` |
| `sleep_on_queue_until_wake_or_timeout` post: returns one of {(), EINTR, EIDRM, EAGAIN} | `Ipc::sleep_on_queue_until_wake_or_timeout` |

### Layer 4: Verus / Creusot functional

Per-`semtimedop(2)` man-page; LTP `ipc/semtimedop*`; SUSv4 conformance.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`semtimedop(2)` reinforcement:

- **Per-timespec validation** — defense against per-overflow-timeout DoS.
- **Per-monotonic clock** — defense against per-wallclock-step bypass.
- **Per-hrtimer-armed only when blocking** — defense against per-fastpath timer alloc.
- All `semop(2)` hardening inherited.

### grsecurity / pax surface

- **PaX UDEREF on copy_from_user(timeout) + copy_from_user(sops)** — defense against per-pointer kernel-deref bug; SMAP forced.
- **PAX_USERCOPY whitelist** — sembuf staging buffer stack-allocated under SEMOPM_FAST limit; otherwise kmalloc'd from a usercopy-allowed slab.
- **GRKERNSEC_HARDEN_IPC** — write/read permission requires euid match cuid OR `CAP_IPC_OWNER` in set's user-ns; same as semop.
- **CAP_IPC_OWNER in set's user-ns strict** — defense against cross-userns sync-bypass.
- **GRKERNSEC_PROC_IPC info-leak** — sempid exposed only to root or set cuid.
- **GRKERNSEC_RAND_RAP / Control Flow Integrity** — hrtimer callback function pointer pinned; defense against per-callback-corruption RCE.
- **Hrtimer leak bounding** — armed timer cancelled on wake/intr/rmid; defense against per-timer-leak.
- **PAX_REFCOUNT on sma + sem_undo** — defense against per-RMID race UAF.
- **PAX_RANDKSTACK** — kstack randomized; defeats ROP gadget chains.
- **IPC namespace strict** — timer wake-queue per-ns; never crosses namespace boundary.
- **GRKERNSEC_AUDIT_IPC** — long-blocking semtimedop emits audit if timeout > configurable threshold (defense against per-DoS via long-blockers).
- **SIGNAL_INTERRUPT POSIX-strict** — no ERESTARTSYS; signal returns EINTR cleanly.
- **No remain-write-out** — defense against per-side-channel info-leak via residual time disclosure.
- **PaX KERNEXEC** — pending_alter / pending_const list pages NX.
- **GRKERNSEC_RANDSTRUCT on sma** — defense against per-heap-spray.

