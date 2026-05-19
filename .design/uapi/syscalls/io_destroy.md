# Tier-5 syscall: io_destroy(2) — syscall 207

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/aio.c (SYSCALL_DEFINE1(io_destroy, ...), ioctx_lookup, free_ioctx_users)
  - include/linux/aio.h
  - include/uapi/linux/aio_abi.h (aio_context_t)
  - arch/x86/entry/syscalls/syscall_64.tbl (207  common  io_destroy)
-->

## Summary

`io_destroy(2)` tears down a Linux AIO (kernel POSIX-style asynchronous I/O) context created by `io_setup(2)`. The context identifier (`aio_context_t`, a `__u64` that is actually the user-space VMA address of the per-context ring buffer) is looked up in the calling task's `mm.ioctx_table`, removed from the table, and its in-flight IOCBs are cancelled. The syscall blocks until all currently-submitted IOCBs that cannot be cancelled have completed; once the context's reference count reaches zero, the ring-buffer VMA is unmapped and the `kioctx` struct is freed.

Critical for: any AIO user (databases — InnoDB, PostgreSQL aio backend; userspace iSCSI / NVMe — fio; FUSE bridges), and as the cleanup half of every successful `io_setup`. The new `io_uring` family supersedes AIO; AIO remains for ABI compatibility.

This Tier-5 covers `io_destroy(2)` on x86_64; the syscall is generic-unistd-equivalent across architectures.

## Signature

```c
int io_destroy(aio_context_t ctx_id);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `ctx_id` | `aio_context_t` (`__u64`) | in | Context identifier returned by a prior `io_setup(2)`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. Context destroyed; ring unmapped; pending IOCBs cancelled or completed. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL`  | `ctx_id` does not refer to an AIO context owned by the calling task's mm. |
| `ENOSYS`  | Kernel compiled with `CONFIG_AIO=n`. |
| `EFAULT`  | Internal consistency check failed (very rare; should not be observed by well-behaved userspace). |

## ABI surface

```text
__NR_io_destroy (x86_64) = 207
__NR_io_destroy (i386)   = 246
__NR_io_destroy (arm64)  = 1   /* part of asm-generic block */

typedef __kernel_ulong_t aio_context_t;   /* opaque; really the user VA of the ring */

Initial state after io_setup:
  ctx->users = 2          /* owner ref + ioctx_table ref */
  ctx->dead  = 0
  ctx->reqs_active = 0

After io_destroy completes:
  ctx removed from mm.ioctx_table
  ctx->dead = 1
  pending IOCBs cancelled (best-effort) or waited-for
  ring-buffer VMA unmapped via munmap-equivalent
  ctx freed once reqs_active reaches 0
```

### Limits

```text
ctx_id          : valid aio_context_t from prior io_setup, in current task's mm
contexts per mm : limited by aio_max_nr (sysctl kernel.aio-max-nr, default 65536)
blocking time   : bounded by completion of un-cancellable IOCBs (no hard deadline)
```

## Compatibility contract

REQ-1: Syscall number is **207** on x86_64. ABI-stable since 2.5.32.

REQ-2: `ctx_id` is looked up in `current->mm->ioctx_table`. If not found, return `-EINVAL`. The lookup is keyed by the user-space ring VA (the value returned from `io_setup`).

REQ-3: A different `mm` (i.e., another process that did NOT inherit this mm via `CLONE_VM`) cannot destroy the context: lookup fails ⟹ `-EINVAL`. This is the access-control gate.

REQ-4: On success: the context is removed from `ioctx_table`, `ctx->dead = 1`, and `ctx->users` is decremented.

REQ-5: Cancel pending IOCBs: for each IOCB on `ctx->active_reqs`, the kernel invokes the file-op `aio_cancel`. IOCBs that support cancel return promptly with `-EINTR` in the completion ring; IOCBs that do not (most block-device direct I/O) are left to complete naturally.

REQ-6: Block until all in-flight IOCBs (`reqs_active`) have either been cancelled or have completed. This is the syscall's primary blocking point.

REQ-7: Once `reqs_active == 0`, the kernel:
- unmaps the ring-buffer VMA (the user-space pages that hold `aio_ring` and the completion events);
- drops the final `ctx->users` reference;
- `free_ioctx` reclaims `kioctx` storage via RCU.

REQ-8: After successful `io_destroy`, any subsequent `io_submit`, `io_getevents`, `io_cancel`, or repeat `io_destroy` referencing the same `ctx_id` returns `-EINVAL` (context is gone from the table).

REQ-9: Per-fork without `CLONE_VM`: the child's `ioctx_table` is freshly allocated empty (AIO contexts do NOT cross fork). The child must call `io_setup` itself.

REQ-10: Per-fork with `CLONE_VM`: the child shares the parent's `mm` and therefore the same `ioctx_table`. Either may call `io_destroy`.

REQ-11: Per-execve: `exit_aio(mm)` is called, which iterates `ioctx_table` and destroys each context (same path as explicit `io_destroy`). New process starts with no AIO contexts.

REQ-12: Per-exit: same as execve — all contexts destroyed.

REQ-13: Per-CONFIG_AIO=n: syscall returns `-ENOSYS`.

REQ-14: Per-ENOMEM avoidance: `io_destroy` itself never allocates. The blocking wait uses a per-ctx completion that is allocated during `io_setup`.

REQ-15: Per-signal: blocking wait is uninterruptible (TASK_UNINTERRUPTIBLE) by design — interrupting the destroy mid-flight would leak the context. The blocking time is bounded by I/O completion (which itself may be capped by device timeouts).

REQ-16: Per-concurrent io_destroy: two threads in the same mm calling `io_destroy(ctx)` race; one succeeds, the other returns `-EINVAL`. Serialized via `ioctx_table.lock`.

REQ-17: Per-statistics: `/proc/sys/kernel.aio_nr` is decremented by the context's max IOCB count when the context is freed.

REQ-18: Per-RCU: the `kioctx` struct is freed under RCU; any in-flight readers (e.g., a concurrent `io_getevents` on a now-destroyed context) safely observe `ctx->dead` and return `-EINVAL`.

## Acceptance Criteria

- [ ] AC-1: `io_setup(N, &ctx)` then `io_destroy(ctx)` returns 0.
- [ ] AC-2: `io_destroy(0xdeadbeef)` (never io_setup'd) returns `-EINVAL`.
- [ ] AC-3: `io_destroy(ctx)` then `io_destroy(ctx)` again returns `-EINVAL` (second call).
- [ ] AC-4: `io_setup(N, &ctx)`; submit 8 IOCBs via `io_submit`; `io_destroy(ctx)` cancels cancellable ones and waits for non-cancellable ones; returns 0.
- [ ] AC-5: After `io_destroy`, any `io_getevents(ctx, ...)` returns `-EINVAL`.
- [ ] AC-6: After `io_destroy`, the ring-buffer VMA is gone (verify via `/proc/self/maps`).
- [ ] AC-7: `/proc/sys/fs/aio-nr` decreases after `io_destroy` by the context's `aio_max_nr_per_ctx`.
- [ ] AC-8: After `execve`, all prior contexts are destroyed; calling `io_destroy(ctx)` post-exec returns `-EINVAL`.
- [ ] AC-9: Two threads sharing the mm calling `io_destroy(ctx)` simultaneously: exactly one returns 0, the other `-EINVAL`.
- [ ] AC-10: `io_destroy` of a context with active non-cancellable I/O blocks until that I/O completes; returns 0.
- [ ] AC-11: Per-`CONFIG_AIO=n` kernel: `io_destroy(any)` returns `-ENOSYS`.
- [ ] AC-12: Per-LTP `io_destroy01..03` pass.

## Architecture

```rust
#[syscall(nr = 207, abi = "sysv")]
pub fn sys_io_destroy(ctx_id: u64) -> isize {
    IoDestroy::dispatch(ctx_id)
}
```

`IoDestroy::dispatch(ctx_id) -> isize`:
1. let mm = current.mm();
2. let ctx = mm.ioctx_table.remove(ctx_id).ok_or(EINVAL)?;
3. /* Mark dead */
4. ctx.dead.store(true, SeqCst);
5. /* Cancel pending */
6. IoDestroy::kill_ioctx_users(&ctx);
7. /* Wait for in-flight */
8. ctx.reqs_active_zero.wait();   /* uninterruptible */
9. /* Unmap ring */
10. Mm::munmap(mm, ctx.user_ring_addr, ctx.ring_pages * PAGE_SIZE);
11. /* Decrement aio_nr */
12. aio_nr_dec(ctx.max_reqs);
13. /* Drop last ref; RCU-free */
14. drop(ctx);
15. Ok(0)

`IoDestroy::kill_ioctx_users(ctx)`:
1. /* Iterate active_reqs */
2. for iocb in ctx.active_reqs.iter() {
3.   if let Some(cancel_fn) = iocb.file.f_op.aio_cancel {
4.     cancel_fn(iocb);
5.   }
6. }
7. /* Wake any sleepers in io_getevents on this ctx */
8. ctx.wait.wake_all();

`exit_aio(mm)`:    /* called by execve / exit */
1. for ctx_id in mm.ioctx_table.ids() {
2.   IoDestroy::dispatch(ctx_id);
3. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lookup_in_current_mm` | INVARIANT | per-dispatch: lookup uses current.mm.ioctx_table only. |
| `not_found_einval` | INVARIANT | per-dispatch: missing ctx ⟹ -EINVAL, no mutation. |
| `dead_after_remove` | INVARIANT | per-dispatch: ok ⟹ ctx.dead == true. |
| `wait_for_reqs_zero` | INVARIANT | per-dispatch: ring unmap happens after reqs_active == 0. |
| `ring_unmapped` | INVARIANT | per-dispatch: ok ⟹ user_ring_addr VMA gone. |
| `aio_nr_dec` | INVARIANT | per-dispatch: ok ⟹ aio_nr -= ctx.max_reqs. |
| `double_destroy_einval` | INVARIANT | per-dispatch: second destroy ⟹ -EINVAL. |

### Layer 2: TLA+

`fs/aio_destroy.tla`:
- States: per-mm ioctx_table; per-ctx dead/users/reqs_active; aio_nr counter.
- Properties:
  - `safety_table_consistent` — ctx in table ⟺ ctx.dead == false.
  - `safety_no_use_after_dead` — io_submit/getevents on dead ctx ⟹ -EINVAL.
  - `safety_reqs_drained` — ring unmap ⟹ reqs_active == 0.
  - `safety_double_destroy` — exactly one destroyer wins.
  - `liveness_terminates` — io_destroy returns once in-flight drains.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `dispatch` post: ctx removed from table; ring unmapped | `IoDestroy::dispatch` |
| `dispatch` post: err ⟹ table unchanged | `IoDestroy::dispatch` |
| `kill_ioctx_users` post: all cancellable IOCBs cancelled | `IoDestroy::kill_ioctx_users` |
| `exit_aio` post: ioctx_table empty | `exit_aio` |

### Layer 4: Verus/Creusot functional

Per-`io_destroy(2)` man-page semantic equivalence. LTP `io_destroy01..03` and `fio --ioengine=libaio` smoke pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`io_destroy(2)` reinforcement:

- **Per-mm scoping of `ioctx_table`** — defense against per-cross-process context destruction.
- **Per-RCU-free of `kioctx`** — defense against per-use-after-free in concurrent `io_getevents`.
- **Per-uninterruptible wait** — defense against per-leak via interrupted destroy.
- **Per-double-destroy `-EINVAL`** — defense against per-double-free of `kioctx`.
- **Per-cancel before wait** — defense against per-indefinite-block via cancellable IOCBs.
- **Per-aio_nr decrement** — defense against per-aio-context-exhaustion (the sysctl limit).
- **Per-ring-VMA unmap** — defense against per-stale-user-ring-mapping leaking event data.
- **Per-execve / per-exit `exit_aio`** — defense against per-process-exit AIO leak.
- **Per-CLONE_VM share semantics** — defense against per-thread-A-destroy-via-thread-B's table.

## Grsecurity / PaX surface

- **PaX UDEREF on internal user pointers** — defense against per-malicious-user-pointer kernel deref bug (the ring-VA derived from ctx_id is validated against the mm's VMA tree).
- **PAX_RANDKSTACK at io_destroy entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_KMEM** — `kioctx` internals never exposed via `/dev/kmem` or `/proc/<pid>/mem`.
- **GRKERNSEC_CHROOT_NO_AIO** — inside grsec chroot, AIO syscalls (including io_destroy) may be denied (policy-controlled); destroy of already-set-up contexts is permitted to allow cleanup.
- **Per-grsec RBAC** — `+I` flag (aio-allow) gates per-subject; default policy permits.
- **Per-grsec audit** — io_destroy of a context with N pending IOCBs logged with N for forensic correlation.
- **PaX REFCOUNT** — `ctx->users` and `ctx->reqs_active` are PAX_REFCOUNT-protected atomic counters; under/over-flow panics.
- **GRKERNSEC_HIDESYM** — `kioctx` struct symbols not leaked via kallsyms or BPF introspection.
- **Per-grsec aio_max_nr-strict** — per-namespace aio_max_nr default lowered (e.g., 1024) to bound exposure; io_destroy correctly returns budget.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `io_setup(2)`, `io_submit(2)`, `io_getevents(2)`, `io_cancel(2)` (separate Tier-5 docs).
- `io_uring` family (separate Tier-3 in `fs/io_uring/00-overview.md`).
- AIO ring-buffer layout (Tier-3 in `fs/aio.md`).
- Block-layer cancellation semantics (Tier-3 in `block/00-overview.md`).
- Implementation code.
