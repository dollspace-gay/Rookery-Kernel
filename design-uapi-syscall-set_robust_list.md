---
title: "Tier-5 syscall: set_robust_list(2) — syscall 273"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`set_robust_list(2)` records, in the task, a user-space pointer to a per-thread linked list of futexes the thread "owns". On thread exit (normal or abnormal), the kernel walks this list and, for each futex, atomically sets `FUTEX_OWNER_DIED` in the futex word and wakes one waiter, allowing PI-mutexes / NPTL robust mutexes to recover from crashed owners without deadlocking other threads.

The syscall takes a pointer to `struct robust_list_head` and a declared size; the kernel validates `len == sizeof(struct robust_list_head)` and records the pointer in `task->robust_list`. It does NOT dereference at call time. Critical for: glibc `pthread_mutex_t` with `PTHREAD_MUTEX_ROBUST`, distributed lock recovery, crash-safe IPC.

### Acceptance Criteria

- [ ] AC-1: `set_robust_list(&head, sizeof head)` returns 0.
- [ ] AC-2: `set_robust_list(&head, 17)` returns -EINVAL.
- [ ] AC-3: `get_robust_list(0, &out, &outlen)` returns the same `&head` and `len`.
- [ ] AC-4: Thread sets robust list, locks a robust mutex (links into list), crashes (SIGKILL): another thread observes `FUTEX_OWNER_DIED` on the mutex.
- [ ] AC-5: Same as AC-4 but waiter (FUTEX_WAITERS bit set) is woken at crash.
- [ ] AC-6: List with 4096 nodes: walk truncates at ROBUST_LIST_LIMIT (2048); no infinite-loop hang.
- [ ] AC-7: `list_op_pending` set during a crash mid-LOCK: kernel marks it OWNER_DIED on exit.
- [ ] AC-8: Compat task (32-bit): `set_robust_list` with sizeof(compat) succeeds; 64-bit size returns -EINVAL.
- [ ] AC-9: After fork, child's robust list is NULL.
- [ ] AC-10: After execve, robust list is NULL.
- [ ] AC-11: Bad pointer at exit-time: silently skipped; no oops.

### Architecture

```rust
#[syscall(nr = 273, abi = "sysv")]
pub fn sys_set_robust_list(head: UserPtr<RobustListHead>, len: usize) -> isize {
    SetRobustList::do_set_robust_list(head, len)
}
```

`SetRobustList::do_set_robust_list(head, len) -> isize`:
1. if len != size_of::<RobustListHead>() { return -EINVAL; }
2. let task = current();
3. task.robust_list.store(head.as_raw(), Ordering::Release);
4. return 0;

`ExitRobustList::on_exit(task)` (called from mm_release):
1. let head_ptr = task.robust_list.swap(null_mut(), Ordering::AcqRel);
2. if head_ptr.is_null() { return; }
3. /* Fetch head fields with fault tolerance. */
4. let head = match fetch_robust_list(head_ptr) { Ok(h) => h, Err(_) => return };
5. /* list_op_pending handled first. */
6. if !head.list_op_pending.is_null() {
7.   let _ = handle_futex_death(head.list_op_pending, head.futex_offset, /*pi=*/false);
8. }
9. /* Walk linked list with bounded depth. */
10. let mut entry = head.list.next;
11. let mut limit = ROBUST_LIST_LIMIT;
12. while !entry.is_null() && entry != &head.list as *const _ && limit > 0 {
13.   let next = match fetch_robust_entry(entry) { Ok(e) => e, Err(_) => break };
14.   if entry != head.list_op_pending {
15.     let _ = handle_futex_death(entry, head.futex_offset, /*pi=*/false);
16.   }
17.   entry = next;
18.   limit -= 1;
19. }

`handle_futex_death(entry, offset, pi) -> Result<()>`:
1. let uaddr = (entry as isize + offset) as *mut u32;
2. loop {
3.   let uval = get_user(uaddr)?;
4.   if (uval & FUTEX_TID_MASK) != task_pid_vnr(current()) { return Ok(()); }
5.   let nval = (uval & FUTEX_WAITERS) | FUTEX_OWNER_DIED;
6.   match futex_atomic_cmpxchg_inatomic(uaddr, uval, nval) {
7.     Ok(prev) if prev == uval => {
8.       if (nval & FUTEX_WAITERS) != 0 {
9.         let _ = do_futex(uaddr, FUTEX_WAKE, 1, None, None, 0);
10.      }
11.      return Ok(());
12.    }
13.    Ok(_) => continue,
14.    Err(_) => return Err(EFAULT),
15.  }
16. }

### Out of Scope

- `get_robust_list(2)` reader counterpart (covered in `get_robust_list.md`).
- PI-futex robust list semantics (covered in `futex.md`).
- glibc NPTL robust mutex implementation.
- Implementation code.

### signature

```c
long set_robust_list(struct robust_list_head *head, size_t len);
```

```c
struct robust_list {
    struct robust_list __user *next;
};

struct robust_list_head {
    struct robust_list        list;
    long                      futex_offset;
    struct robust_list __user *list_op_pending;
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `head` | `struct robust_list_head *` | in (recorded) | Per-thread list anchor in user memory. Layout fixed by ABI. May be NULL to clear. |
| `len` | `size_t` | in | Must equal `sizeof(struct robust_list_head)`; serves as forward-compat ABI check. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; pointer recorded. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `len != sizeof(struct robust_list_head)`. |

`set_robust_list(2)` does not validate `head` (no `access_ok`/`copy_from_user`); faults on the list at exit-time are caught and silently aborted.

### abi surface

```text
__NR_set_robust_list  (x86_64) = 273
__NR_set_robust_list  (arm64)  = 99
__NR_set_robust_list  (riscv)  = 99
__NR_set_robust_list  (i386)   = 311  (compat: 32-bit struct robust_list_head)

/* 32-bit compat layout differs: pointers are 32 bits.
   The compat handler set_robust_list32 enforces
   len == sizeof(struct compat_robust_list_head32). */
```

### compatibility contract

REQ-1: Syscall number is **273** on x86_64. ABI-stable.

REQ-2: `len` must equal `sizeof(struct robust_list_head)` (24 bytes on 64-bit, 12 bytes on 32-bit compat). Otherwise `-EINVAL`.

REQ-3: `head` is stored in `current->robust_list` verbatim; no validation at call time.

REQ-4: A previous robust list pointer is overwritten; the old list is not unregistered (user owns the lifecycle).

REQ-5: Per-`exit_robust_list(task)` is invoked from `mm_release()` on task exit:
- Reads `head` via fetch_robust_list (user copy).
- Walks `head.list.next` chain.
- For each list node, computes futex address = `node + head.futex_offset` (offset is signed).
- Atomically `or`s `FUTEX_OWNER_DIED` into futex word.
- If `FUTEX_WAITERS` set, `do_futex(FUTEX_WAKE, 1)` on the address.
- Limits chain walk to `ROBUST_LIST_LIMIT = 2048` to prevent infinite loops.

REQ-6: `head.list_op_pending` allows recovery of a partially-acquired futex (between LOCK and link-into-list, or between unlink and UNLOCK). Kernel processes it once before the main chain walk.

REQ-7: 32-bit compat (`compat_set_robust_list`): separate `compat_robust_list_head` with 32-bit pointers; size check uses compat size.

REQ-8: Fork: child's `robust_list` is set to NULL (per-thread list, not inherited).

REQ-9: Execve: mm replaced; new mm has no robust list. `robust_list` reset to NULL.

REQ-10: PID namespace agnostic: address is virtual in caller's mm; no translation needed.

REQ-11: `set_robust_list(NULL, 0)` is invalid because `len` must match `sizeof(robust_list_head)`; to clear, call `set_robust_list(NULL, sizeof(struct robust_list_head))`.

REQ-12: Reading the list at exit uses `get_user` / `futex_atomic_cmpxchg_inatomic`; pagefaults are tolerated by retry, but unrecoverable faults abort that node and proceed.

### abi surface (struct)

```c
#define ROBUST_LIST_LIMIT 2048
#define FUTEX_OWNER_DIED  0x40000000
#define FUTEX_WAITERS     0x80000000
```

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `len_size_check` | INVARIANT | len != sizeof(RobustListHead) ⟹ EINVAL. |
| `head_stored_verbatim` | INVARIANT | task.robust_list == arg.head after call. |
| `walk_bounded` | INVARIANT | exit walk terminates within ROBUST_LIST_LIMIT iterations. |
| `owner_died_atomic` | INVARIANT | each futex word receives OR FUTEX_OWNER_DIED via cmpxchg. |
| `waiters_woken_iff_bit_set` | INVARIANT | FUTEX_WAKE issued iff FUTEX_WAITERS was set. |

### Layer 2: TLA+

`kernel/robust-list.tla`:
- States: per-task robust_list pointer, per-list chain, per-futex word.
- Properties:
  - `safety_size_check` — only sizeof match succeeds.
  - `safety_walk_bounded` — chain walk halts within 2048.
  - `safety_owner_died_set` — every traversed futex marked OWNER_DIED.
  - `safety_no_inherit_on_fork` — child task.robust_list == NULL.
  - `liveness_exit_completes` — exit_robust_list terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_set_robust_list` post: len mismatch ⟹ EINVAL | `SetRobustList::do_set_robust_list` |
| `do_set_robust_list` post: success ⟹ task.robust_list == arg | `SetRobustList::do_set_robust_list` |
| `exit_robust_list` post: all owned futexes marked OWNER_DIED | `ExitRobustList::on_exit` |
| `handle_futex_death` post: futex updated via cmpxchg, waiters woken | `handle_futex_death` |

### Layer 4: Verus/Creusot functional

Per-`set_robust_list(2)` / `get_robust_list(2)` man pages + Documentation/locking/robust-futexes.txt semantic equivalence. glibc `pthread_mutex_t` with `PTHREAD_MUTEX_ROBUST` survives owner crash.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`set_robust_list(2)` reinforcement:

- **Per-len exact-size check** — defense against per-extension-field smuggling.
- **Per-walk bounded by ROBUST_LIST_LIMIT** — defense against per-malicious-list infinite-loop DoS.
- **Per-fault-on-fetch silently aborts current node** — defense against per-crash-corrupted-list kernel oops.
- **Per-atomic cmpxchg on futex word** — defense against per-races with other threads.
- **Per-TID-match before OWNER_DIED set** — defense against per-spoofed-ownership across recycled TIDs.
- **Per-fork explicit reset** — defense against per-child inheriting parent's list head.

### grsecurity / pax surface

- **PaX UDEREF on exit-time fetch_robust_list** — defense against per-exit kernel-deref bug; SMAP forced.
- **Robust-list MMU guarantee** — the kernel must verify that the recorded head pointer is in the caller's mm at exit time (mm not detached) before any user access; grsec adds an mm-validity assertion to fetch_robust_list so a freed mm cannot be walked.
- **PAX_USERCOPY_HARDEN on per-node get_user** — bounded 8-byte reads on whitelisted user mappings.
- **GRKERNSEC_HIDESYM on `task->robust_list`** — pointer not exposed via `/proc/$pid/status` to non-CAP_SYS_PTRACE.
- **PAX_REFCOUNT on robust-list traversal counter** — defense against per-counter-overflow widening the 2048 cap.
- **GRKERNSEC_FUTEX_KEY_PRIVATE** — exit-time wakes forced FUTEX_PRIVATE_FLAG so cross-process attackers cannot collide keys.
- **Per-cred snapshot at exit** — defense against per-cred-elevation between list-register and exit-walk.
- **Per-PaX MPROTECT tolerance** — write-disabled mappings tolerated; node skipped, no oops.
- **Per-list_op_pending strict bound to one entry** — defense against per-pending-cycle exploit.

