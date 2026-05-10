---
title: "Tier-2: kernel/futex — futex(2) syscall (core + waitwake + requeue + PI + syscall dispatch)"
tags: ["tier-2", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for the `futex(2)` family of syscalls — Linux's userspace-fast-path locking primitive underneath glibc/musl `pthread_mutex` + `pthread_cond` + `sem_post/sem_wait`, Rust's `std::sync::Mutex` + `Condvar`, every userspace lock library + every userspace queue implementation. Components: **core** (`core.c` + `futex.h`: `futex_q` waiter management, hashed bucket lookup, `get_futex_key` for shared/private + cross-mm support, RCU-protected hash bucket walks), **waitwake** (`waitwake.c`: `FUTEX_WAIT` / `_WAIT_BITSET` / `_WAKE` / `_WAKE_BITSET` / `_WAIT_MULTIPLE`), **requeue** (`requeue.c`: `FUTEX_CMP_REQUEUE` / `_REQUEUE_PI` — used by glibc condvar broadcast to avoid thundering-herd), **PI** (`pi.c`: priority-inheritance futex `FUTEX_LOCK_PI` / `_LOCK_PI2` / `_UNLOCK_PI` / `_TRYLOCK_PI` — used by RT pthread mutexes), **syscalls** (`syscalls.c`: futex(2) syscall #202 + futex_waitv(2) syscall #449 + futex_wake(2) #454 + futex_wait(2) #455 + futex_requeue(2) #456 dispatch).

`futex_waitv` (vectored wait, added recently) lets a thread atomically wait on N futexes simultaneously — backbone of WINE/Proton's NTSync-replacement.

### Acceptance Criteria

- [ ] AC-O1: glibc `nptl-stress-test` suite passes (covers pthread_mutex + cond_wait + sem).
- [ ] AC-O2: Rust `std::sync` test suite passes.
- [ ] AC-O3: Robust-list test: program holding futex SIGKILL'd; next claimer sees FUTEX_OWNER_DIED bit + acquires.
- [ ] AC-O4: PI test: high-prio thread blocks on PI-futex held by low-prio; low-prio inherits boost; verified via /proc/<pid>/sched.
- [ ] AC-O5: futex_waitv test: WINE NTSync workload runs at expected throughput.
- [ ] AC-O6: kselftest `tools/testing/selftests/futex/` passes.

### Out of Scope

- Implementation code
- 32-bit-only paths
- futex_fd (FUTEX_FD op was deprecated long ago — not implemented)

### compatibility contract — outline

- `SYS_futex` (#202) ABI byte-identical: `op` argument values FUTEX_WAIT / WAKE / FD (deprecated) / REQUEUE / CMP_REQUEUE / WAKE_OP / LOCK_PI / UNLOCK_PI / TRYLOCK_PI / WAIT_BITSET / WAKE_BITSET / WAIT_REQUEUE_PI / CMP_REQUEUE_PI / LOCK_PI2 with FUTEX_PRIVATE_FLAG + FUTEX_CLOCK_REALTIME modifiers.
- `SYS_futex_waitv` (#449), `SYS_futex_wake` (#454), `SYS_futex_wait` (#455), `SYS_futex_requeue` (#456) ABI byte-identical (modern unified replacement family).
- Per-op semantics + per-op return values byte-identical (every glibc / musl test passes).
- Robust list (`set_robust_list(2)` + `get_robust_list(2)`) byte-identical so process-death cleanup of held futexes works.
- `FUTEX_OWNER_DIED` bit + `FUTEX_TID_MASK` semantics for PI futexes byte-identical.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `kernel/futex/core.md` | `core.c` + `futex.h`: futex_q + hashed bucket + get_futex_key + RCU walks |
| `kernel/futex/waitwake.md` | `waitwake.c`: WAIT/WAKE/WAIT_BITSET/WAKE_BITSET |
| `kernel/futex/requeue.md` | `requeue.c`: CMP_REQUEUE + REQUEUE_PI (glibc condvar broadcast) |
| `kernel/futex/pi.md` | `pi.c`: priority-inheritance LOCK_PI/UNLOCK_PI/TRYLOCK_PI/LOCK_PI2 |
| `kernel/futex/syscalls.md` | `syscalls.c`: per-syscall dispatch (futex/futex_waitv/futex_wake/futex_wait/futex_requeue) |
| `kernel/futex/robust-list.md` | `core.c`: robust list + process-death cleanup |

### compatibility outline

- REQ-O1: All futex syscall numbers + arg ABI + per-op semantics byte-identical (glibc / musl / Rust std test suites pass).
- REQ-O2: Robust list semantics identical (process-death cleanup works).
- REQ-O3: PI futex priority inheritance correctness preserved.
- REQ-O4: futex_waitv vectored-wait identical (WINE/Proton NTSync replacement works).
- REQ-O5: TLA+ models (futex_q queue + wake-up race-freedom; PI proxy-locking deadlock avoidance via wait-die; requeue atomicity).
- REQ-O6: Verus/Creusot Layer-4 contracts on hash-bucket lookup correctness + per-key futex_q ownership invariant.
- REQ-O7: Hardening: per-cgroup futex isolation (CONFIG_FUTEX_PI gated for RT); robust-list size cap; user-supplied addr validated for VMA bounds before atomic ops.

### verification

| TLA+ Model | Owner |
|---|---|
| `models/futex/wait_wake.tla` | `kernel/futex/waitwake.md` (proves: FUTEX_WAIT + concurrent FUTEX_WAKE never produce lost wakeup; per-bucket lock + per-key match correct under concurrent waiters/wakers; spurious-wake handled correctly) |
| `models/futex/pi_chain.tla` | `kernel/futex/pi.md` (proves: PI proxy-locking with N-deep priority chain converges; cycle detection prevents deadlock; PI boost/unboost paired correctly) |
| `models/futex/requeue_atomic.tla` | `kernel/futex/requeue.md` (proves: FUTEX_CMP_REQUEUE atomic compare + requeue across two futexes; concurrent waker on source futex during requeue serialized correctly) |
| `models/futex/robust_list.tla` | `kernel/futex/robust-list.md` (proves: per-task robust list + process-exit cleanup; partially-held futex on exit cleared with OWNER_DIED bit; concurrent claim during exit observes correct value) |

Layer-4: `core.md` — `get_futex_key(uaddr, fshared, &key)` post: returned key uniquely identifies the (mm, page, offset) for private OR (inode, page-offset) for shared; concurrent mremap/munmap handled correctly via VMA refcount.

### hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-key inode/file refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-op dispatch table `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-bucket waiter-count + futex_waitv array length checked | § Mandatory |
| **MEMORY_SANITIZE** | freed futex_q state cleared (carries waiter task ptr) | § Default-on configurable |

Futex-specific reinforcement: per-cgroup futex_pi isolation — cross-cgroup PI boost can leak scheduling info; mediated via cgroup-v2 RT controller. Robust-list size cap (`/proc/sys/kernel/robust_list_max=`, default 1M entries). User-supplied uaddr validated for VMA bounds + accessibility before atomic CAS (defense against fault-during-CAS leaking kernel state). futex_waitv vector-len capped at FUTEX_WAITV_MAX (128) — defense against arbitrary vector-size attack.

