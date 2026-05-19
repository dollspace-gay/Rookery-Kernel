# Tier-5 syscall: get_robust_list(2) — syscall 274

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/futex/syscalls.c (SYSCALL_DEFINE3(get_robust_list))
  - kernel/futex/core.c (exit_robust_list)
  - include/uapi/linux/futex.h (struct robust_list_head)
  - arch/x86/entry/syscalls/syscall_64.tbl (274 common get_robust_list)
-->

## Summary

`get_robust_list(2)` is the reader counterpart of `set_robust_list(2)`. Given a target `pid` (or `0` for self), it returns through user-space output parameters the address of the target's per-thread robust-list head and the declared size. Capability gating allows a process to read any task whose credentials match (or with `CAP_SYS_PTRACE`), making this primarily a debugger / `ptrace`-tool primitive used by gdb, perf, and lock-debugging utilities to recover the robust-mutex state of a crashed thread.

Critical for: gdb robust-mutex inspection, ptrace-based introspection, post-mortem lock analysis, sched_ext debuggers.

## Signature

```c
long get_robust_list(int pid,
                     struct robust_list_head **head_ptr,
                     size_t *len_ptr);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pid` | `int` | in | Target TID (kernel `pid_t`); `0` = current. |
| `head_ptr` | `struct robust_list_head **` | out (user) | Receives the recorded head pointer (a user-space address valid in target's mm). |
| `len_ptr` | `size_t *` | out (user) | Receives `sizeof(struct robust_list_head)` (always the kernel's compile-time size for the target's ABI). |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; `*head_ptr` and `*len_ptr` written. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `ESRCH` | No task with TID == `pid`. |
| `EPERM` | Caller lacks permission to inspect target (cred mismatch and no `CAP_SYS_PTRACE` in target's user namespace). |
| `EFAULT` | `head_ptr` or `len_ptr` not writable. |

## ABI surface

```text
__NR_get_robust_list  (x86_64) = 274
__NR_get_robust_list  (arm64)  = 100
__NR_get_robust_list  (riscv)  = 100
__NR_get_robust_list  (i386)   = 312  (compat: 32-bit pointers)

/* sizeof(struct robust_list_head) is 24 on 64-bit, 12 on 32-bit. */
```

## Compatibility contract

REQ-1: Syscall number is **274** on x86_64. ABI-stable.

REQ-2: `pid == 0` means current task. Else `find_task_by_vpid(pid)` in caller's PID namespace.

REQ-3: Permission check: caller's `cred` matches target's, OR caller has `CAP_SYS_PTRACE` in target's `task_active_pid_ns()->user_ns`. Mirrors `ptrace_may_access(target, PTRACE_MODE_READ_REALCREDS)`.

REQ-4: Output via `put_user`:
- `*head_ptr` = `target->robust_list` (user-space pointer in target's mm).
- `*len_ptr` = `sizeof(struct robust_list_head)`.

REQ-5: 32-bit compat path (`compat_get_robust_list`):
- `len_ptr` receives `sizeof(struct compat_robust_list_head)` = 12.
- `head_ptr` is `compat_uptr_t` (32-bit pointer); high bits zero.

REQ-6: Target running with NULL robust_list returns `*head_ptr = NULL`, `*len_ptr = sizeof(...)`, status 0.

REQ-7: After target exit (but before reap by parent), `task_struct` still exists; `target->robust_list` already cleared by `exit_robust_list`; returns NULL.

REQ-8: After target reap (`task_struct` released), `pid` lookup returns NULL ⟹ `-ESRCH`.

REQ-9: `head_ptr` user pointer is in **caller's** mm; not the target's. Kernel translates to caller's address space for the `put_user`.

REQ-10: The value written into `*head_ptr` is a user-space VA from the **target's** mm; the caller is expected to use `process_vm_readv(2)` or `ptrace(PTRACE_PEEKDATA, ...)` to dereference it.

REQ-11: rcu_read_lock + get_task_struct around target deref; no sleep with rcu.

REQ-12: No locking of target's `task_struct.robust_list`; it is set via atomic store and read here via plain load — best-effort snapshot.

## Acceptance Criteria

- [ ] AC-1: `get_robust_list(0, &h, &l)` returns 0, `l = sizeof(struct robust_list_head)`, `h` = previously-set pointer.
- [ ] AC-2: After `set_robust_list(&head, sizeof head)`, `get_robust_list(gettid(), ...)` returns the same `&head`.
- [ ] AC-3: `get_robust_list(99999999, ...)` (nonexistent TID) returns `-ESRCH`.
- [ ] AC-4: Non-root reading another user's task: returns `-EPERM`.
- [ ] AC-5: Root reading any task: returns 0.
- [ ] AC-6: `head_ptr = NULL` (bad output): returns `-EFAULT`.
- [ ] AC-7: 32-bit caller of 32-bit task: `*len_ptr` = 12; pointer is 32-bit zero-extended.
- [ ] AC-8: Target whose `robust_list == NULL`: returns 0 with `*head_ptr = NULL`.
- [ ] AC-9: Target exiting (zombie): returns 0 with `*head_ptr = NULL` (already cleared).
- [ ] AC-10: ptrace-allowed across user-ns boundary: returns 0; otherwise `-EPERM`.

## Architecture

```rust
#[syscall(nr = 274, abi = "sysv")]
pub fn sys_get_robust_list(pid: i32,
                            head_ptr: UserPtr<*mut RobustListHead>,
                            len_ptr: UserPtr<usize>) -> isize {
    GetRobustList::do_get_robust_list(pid, head_ptr, len_ptr)
}
```

`GetRobustList::do_get_robust_list(pid, head_ptr, len_ptr) -> isize`:
1. let target: Arc<Task>;
2. if pid == 0 {
3.   target = current();
4. } else {
5.   rcu_read_lock();
6.   let t = find_task_by_vpid(pid).ok_or(ESRCH)?;
7.   if !ptrace_may_access(&t, PTRACE_MODE_READ_REALCREDS) {
8.     rcu_read_unlock();
9.     return -EPERM;
10.  }
11.  target = get_task_struct(t);
12.  rcu_read_unlock();
13. }
14. let head = target.robust_list.load(Ordering::Acquire);
15. let len = size_of::<RobustListHead>();
16. head_ptr.put(head).map_err(|_| EFAULT)?;
17. len_ptr.put(len).map_err(|_| EFAULT)?;
18. put_task_struct(target);
19. return 0;

`Compat::do_get_robust_list32(pid, head_ptr, len_ptr) -> isize`:
1. /* Same as 64-bit but writes 32-bit pointer and 32-bit length. */
2. let head32 = (target.compat_robust_list as u32);
3. head_ptr.put(head32)?;
4. len_ptr.put(size_of::<CompatRobustListHead>() as u32)?;

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `target_lookup_or_esrch` | INVARIANT | pid != 0 ∧ not-found ⟹ ESRCH. |
| `perm_check_before_read` | INVARIANT | unauthorized ⟹ EPERM, no robust_list read. |
| `head_value_consistent` | INVARIANT | put_user(head) writes exactly target.robust_list. |
| `len_value_constant` | INVARIANT | put_user(len) writes sizeof(RobustListHead). |
| `efault_on_bad_outptr` | INVARIANT | put_user failure ⟹ EFAULT. |

### Layer 2: TLA+

`kernel/get-robust-list.tla`:
- States: per-target task table, per-cred check, per-out-pointer write.
- Properties:
  - `safety_esrch_on_missing` — nonexistent pid ⟹ ESRCH.
  - `safety_eperm_unprivileged_cross_cred` — non-matching cred ∧ no CAP_SYS_PTRACE ⟹ EPERM.
  - `safety_correct_head_returned` — *head_ptr == target.robust_list.
  - `safety_correct_len_returned` — *len_ptr == sizeof(RobustListHead).
  - `liveness_call_returns` — bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_get_robust_list` post: ret == 0 ⟹ both put_users succeeded | `GetRobustList::do_get_robust_list` |
| `do_get_robust_list` post: ESRCH iff target not found | `GetRobustList::do_get_robust_list` |
| `do_get_robust_list` post: EPERM iff ptrace_may_access denied | `GetRobustList::do_get_robust_list` |
| `do_get_robust_list` post: get_task_struct/put_task_struct balanced | `GetRobustList::do_get_robust_list` |

### Layer 4: Verus/Creusot functional

Per-`get_robust_list(2)` man page + glibc `pthread_getrobustlist_np` semantic equivalence: gdb-style robust-list inspection.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`get_robust_list(2)` reinforcement:

- **Per-ptrace permission gate** — defense against per-info-leak of arbitrary user task's robust list pointer.
- **Per-rcu read lock around find_task_by_vpid** — defense against per-task UAF during lookup.
- **Per-get_task_struct/put_task_struct refcount** — defense against per-task vanish mid-call.
- **Per-put_user failure rolls back to EFAULT** — defense against per-partial-output state.
- **Per-no kernel deref of target's user pointer** — defense against per-cross-mm kernel-deref.
- **Per-32-bit compat zero-extend** — defense against per-pointer high-bit smuggling.

## Grsecurity / PaX surface

- **PaX UDEREF on put_user(head, len)** — defense against per-output kernel-deref bug; SMAP forced.
- **Robust-list MMU guarantee** — even when reporting another task's pointer, the kernel does not dereference into the target's mm; grsec asserts that returned address is opaque and never resolved by the kernel on behalf of the caller.
- **GRKERNSEC_HIDESYM cross-process** — non-root callers always get EPERM, even for `pid == gettid()` of a same-uid task in a different mount-ns, when GRKERNSEC_PROC_USER is set.
- **CAP_SYS_PTRACE in target's user_ns required** — capability check uses `task_active_pid_ns()->user_ns` (not caller's) to prevent userns-escape inspection.
- **PAX_REFCOUNT on task_struct refcount** — defense against per-refcount-overflow UAF during lookup.
- **GRKERNSEC_PROC_ADD restriction** — when GRKERNSEC_PROC_USER restricts /proc, this syscall mirrors the restriction so it cannot be used as an out-of-band ptrace bypass.
- **Per-cred snapshot at entry** — defense against per-cred-elevation between permission check and put_user.
- **Per-compat-mode strict zero-extend** — defense against per-32-bit-pointer truncation oracle.
- **PAX_USERCOPY_HARDEN on output writes** — bounded sizeof pointer + sizeof size_t whitelisted.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `set_robust_list(2)` writer counterpart (covered in `set_robust_list.md`).
- Futex internals (covered in `futex.md`).
- ptrace_may_access decision logic (covered in `ptrace.md`).
- Implementation code.
