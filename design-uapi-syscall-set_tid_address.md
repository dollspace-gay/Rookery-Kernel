---
title: "Tier-5 syscall: set_tid_address(2) — syscall 218"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`set_tid_address(2)` records a user-space pointer in `task_struct::clear_child_tid`. When the task subsequently exits (and was the last user of its mm), the kernel writes `0` to that address and issues a `FUTEX_WAKE` on it, allowing libc (glibc/musl) to implement `pthread_join` without polling. It is also the kernel-side companion to `clone(CLONE_CHILD_CLEARTID)`, but as a standalone syscall it lets a thread update or set the pointer post-creation.

The call is unconditional and always succeeds: it returns the caller's TID. There is no kernel-side validation of the pointer at call time; faults are detected and silently ignored at exit-time. Critical for: NPTL thread joinability, musl thread cleanup, runtime-thread library bootstrap.

### Acceptance Criteria

- [ ] AC-1: `set_tid_address(&t)` returns `gettid()`.
- [ ] AC-2: Calling thread exits: `t` contains `0` (observed by joiner).
- [ ] AC-3: A second thread waiting via `FUTEX_WAIT(&t, expected)` is woken at exit.
- [ ] AC-4: `set_tid_address(NULL)` returns TID and disables the cleartid behavior.
- [ ] AC-5: `set_tid_address((int *)0xdeadbeef)` returns TID and does not fault the caller; at exit, the bad address is silently dropped.
- [ ] AC-6: `clone(CLONE_CHILD_CLEARTID, ..., &t)` then child `set_tid_address(&t2)`: at child exit, `t2` (not `t`) is zeroed.
- [ ] AC-7: After `fork()`, child's `clear_child_tid` is `NULL` regardless of parent's setting.
- [ ] AC-8: After `execve()`, mm-replaced; new mm has no recorded clear_child_tid.
- [ ] AC-9: Multi-thread process where one thread exits but mm-users > 1: no zero-write occurs (only final mm user triggers it).
- [ ] AC-10: TID returned matches `getpid()` for single-threaded process.

### Architecture

```rust
#[syscall(nr = 218, abi = "sysv")]
pub fn sys_set_tid_address(tidptr: UserPtr<i32>) -> isize {
    SetTidAddress::do_set_tid_address(tidptr)
}
```

`SetTidAddress::do_set_tid_address(tidptr) -> isize`:
1. let task = current();
2. task.clear_child_tid.store(tidptr.as_raw(), Ordering::Relaxed);
3. let tid = task_pid_vnr(&task);
4. return tid as isize;

`MmRelease::on_exit(task, mm)` (called from `do_exit` via `mm_release`):
1. let ptr = task.clear_child_tid.swap(null_mut(), Ordering::AcqRel);
2. if ptr.is_null() { return; }
3. if mm.mm_users.load(Ordering::Acquire) != 1 { return; }   // only on final mm user
4. /* Best-effort zero + wake; ignore faults. */
5. let _ = UserPtr::<i32>::from_raw(ptr).put(0);
6. let _ = futex::do_futex(ptr, FUTEX_WAKE, 1, None, None, 0);

### Out of Scope

- `clone(2)` CLONE_CHILD_CLEARTID handling (covered in `clone.md` / `clone3.md`).
- Futex internals (covered in `futex.md`).
- pthread library implementation.
- Implementation code.

### signature

```c
pid_t set_tid_address(int *tidptr);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `tidptr` | `int *` | in (recorded) | User-space address; will receive `0` and a `FUTEX_WAKE` at task exit. May be `NULL` to clear. Not dereferenced at call time. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | The caller's TID (`task->pid`, i.e. kernel `pid_t`). |

`set_tid_address(2)` never fails in the architectural sense; the return type is `pid_t` but the call is documented as always-successful.

### errors

| errno | Trigger |
|---|---|
| (none) | No documented failure path; pointer is recorded as-is, never validated at call time. |

### abi surface

```text
__NR_set_tid_address  (x86_64) = 218
__NR_set_tid_address  (arm64)  = 96
__NR_set_tid_address  (riscv)  = 96
__NR_set_tid_address  (i386)   = 258

/* tidptr is opaque to the kernel until task exit. */
/* glibc stores &THREAD_SELF->tid here at thread-create time. */
```

### compatibility contract

REQ-1: Syscall number is **218** on x86_64. ABI-stable.

REQ-2: Stores `tidptr` into `current->clear_child_tid` unconditionally; previous value is overwritten and not freed (it is a user pointer).

REQ-3: Returns `task_pid_vnr(current)` — the TID as seen from the caller's PID namespace.

REQ-4: No `copy_from_user` / `access_ok` check at call time. Validation happens lazily at exit-time inside `mm_release()`.

REQ-5: At task exit, if `tsk->clear_child_tid != NULL` AND mm has only one user (`atomic_read(&mm->mm_users) == 1` -- final exit on this mm), the kernel:
- `put_user(0, tsk->clear_child_tid)` -- ignored on EFAULT.
- `do_futex(tsk->clear_child_tid, FUTEX_WAKE, 1, NULL, NULL, 0)` -- ignored on error.
- Clears `tsk->clear_child_tid`.

REQ-6: A clone with `CLONE_CHILD_CLEARTID` sets `clear_child_tid` to the `child_tidptr` argument; `set_tid_address(2)` allows post-clone modification of that pointer.

REQ-7: `CLONE_PARENT_SETTID` and `CLONE_CHILD_SETTID` write the TID at clone-time and are unrelated to the cleartid pointer.

REQ-8: PID namespace: the returned TID is the vnr in the caller's namespace; the cleartid futex wake uses the user-space pointer (no namespace translation -- it is a virtual address in the caller's mm).

REQ-9: Across `execve(2)`, `clear_child_tid` is preserved if the new image is in the same mm (it is not -- execve replaces mm), so effectively cleared by the new mm.

REQ-10: Across `fork(2)`, child does NOT inherit `clear_child_tid` (it is per-task and explicitly reset).

REQ-11: The futex wake at exit is `FUTEX_WAKE_PRIVATE`-equivalent semantics: same-process wake; uses mm-relative key.

REQ-12: If `tidptr` is unaligned or unmapped at exit, the `put_user`/`do_futex` calls silently fail; libc must tolerate (joining thread will spin on the address or block until reaper notices via wait4).

REQ-13: `task->clear_child_tid` is a per-task user pointer; the field is plain `int __user *` (not refcounted, not validated). The kernel never holds a reference into the user mm via this pointer; it is dereferenced at most once at exit.

REQ-14: Concurrency: a second thread in the same mm calling `set_tid_address(2)` from the caller's perspective updates only its own `task_struct.clear_child_tid`. There is no shared state across threads in this syscall.

REQ-15: Across `ptrace(2)` attach, the tracer cannot read or write `task->clear_child_tid` directly through the syscall interface; introspection requires `/proc/$pid/...` (and CAP_SYS_PTRACE), but no such file currently exposes it -- the field is effectively kernel-private apart from its self-write semantics.

REQ-16: Per-PID-namespace TID translation is applied to the **return value** (vnr) but NOT to the recorded pointer (which is a virtual address in the caller's mm, namespace-agnostic).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `store_is_total` | INVARIANT | per-call: clear_child_tid store always succeeds. |
| `tid_return_in_caller_ns` | INVARIANT | returned TID equals task_pid_vnr(current). |
| `no_user_access_at_call_time` | INVARIANT | no copy_from_user/copy_to_user during set_tid_address. |
| `exit_clear_on_final_user_only` | INVARIANT | per-mm_release: zero+wake only when mm_users == 1. |
| `fork_resets_pointer` | INVARIANT | child task.clear_child_tid == null after copy_process. |

### Layer 2: TLA+

`kernel/set-tid-address.tla`:
- States: per-task clear_child_tid pointer, per-mm users counter, per-task exit state.
- Properties:
  - `safety_tid_returned` — call returns vnr TID.
  - `safety_clear_only_on_final_exit` — zero+wake gated on mm_users == 1.
  - `safety_fork_no_inherit` — child cleared.
  - `liveness_exit_eventually_wakes` — if pointer valid AND final mm user, futex wake observed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_set_tid_address` post: task.clear_child_tid == arg | `SetTidAddress::do_set_tid_address` |
| `do_set_tid_address` post: ret == task_pid_vnr(current) | `SetTidAddress::do_set_tid_address` |
| `mm_release` post: clear_child_tid == NULL after exit | `MmRelease::on_exit` |
| `mm_release` post: futex_wake invoked iff mm_users == 1 prior | `MmRelease::on_exit` |

### Layer 4: Verus/Creusot functional

Per-`set_tid_address(2)` man page + glibc / musl `pthread_create` / `pthread_join` semantic equivalence: `THREAD_SELF->tid` zeroed and joiner woken at exit.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`set_tid_address(2)` reinforcement:

- **Per-pointer recorded as-is, validated lazily** — defense against per-call DoS via repeated bad pointers (no kernel work at call time).
- **Per-exit zero+wake best-effort, faults ignored** — defense against per-exit kernel panic on bad pointer.
- **Per-final-mm-user gate** — defense against per-thread-group spurious wake of detached threads.
- **Per-task pointer cleared atomically at exit start** — defense against per-double-clear.
- **Per-fork explicit reset** — defense against per-child inheriting parent's joiner address.

### grsecurity / pax surface

- **PaX UDEREF on exit-time put_user** — defense against per-exit kernel-deref bug; SMAP forced even on best-effort path.
- **PAX_USERCOPY_HARDEN on `put_user(0, tidptr)`** — bounded 4-byte write to user mm; whitelisted access pattern.
- **GRKERNSEC_HIDESYM on `task->clear_child_tid` exposure** — pointer not leaked via `/proc/$pid/status` or ptrace gadgets to non-CAP_SYS_PTRACE callers.
- **PAX_REFCOUNT on `mm_users`** — defense against per-mm_users underflow that would mis-trigger the cleartid path.
- **GRKERNSEC_FUTEX_KEY_PRIVATE** — clear_child_tid futex wake forced FUTEX_PRIVATE_FLAG semantics so the key cannot collide with a cross-process attacker.
- **Per-task cred snapshot at exit** — defense against per-cred-elevation during the brief window between clear_child_tid store and exit-time wake.
- **Per-PaX MPROTECT compatibility** — exit-time write tolerates MPROTECT-stripped writability without faulting reaper.
- **GRKERNSEC_PROC_HIDE on `task->clear_child_tid`** — field never serialised into any /proc artefact even for CAP_SYS_PTRACE; defense against per-NPTL-fingerprinting from layout of a hardened binary's thread-control block.
- **PAX_REFCOUNT on `mm->mm_users` at exit-time check** — the mm_users == 1 final-user gate uses a saturating refcount; defense against per-refcount-underflow that would mis-fire the cleartid write on a non-final exit.

