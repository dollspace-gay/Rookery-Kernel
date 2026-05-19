# Tier-3: drivers/dma-buf/dma-fence.c — DMA fence primitive

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/dma-buf/00-overview.md
upstream-paths:
  - drivers/dma-buf/dma-fence.c (~1208 lines)
  - drivers/dma-buf/dma-fence-array.c (composite OR-array, separate Tier-3)
  - drivers/dma-buf/dma-fence-chain.c (composite timeline-chain, separate Tier-3)
  - drivers/dma-buf/dma-fence-unwrap.c (composite iteration, separate Tier-3)
  - include/linux/dma-fence.h
-->

## Summary

A **dma-fence** is a one-shot cross-driver synchronization primitive that represents the completion of an asynchronous DMA operation (GPU rendering, video codec frame, display flip, RDMA transfer, ...). It carries `(context, seqno)` ordering, an RCU-protected `dma_fence_ops` vtable (`get_driver_name`, `get_timeline_name`, `enable_signaling`, `signaled`, `wait`, `release`, `set_deadline`), a callback list (`cb_list`), a refcount (`kref`), flags (`SIGNALED`, `ENABLE_SIGNAL`, `TIMESTAMP`, `SEQNO64`, `INITIALIZED`, `INLINE_LOCK`), and either an inline or driver-provided spinlock. Once `dma_fence_signal()` runs all queued callbacks and sets the SIGNALED bit, the fence stays signaled forever; waiters return immediately. The signalling-critical section is annotated with `dma_fence_begin_signalling()` / `_end_signalling()` so lockdep can validate the cross-driver lock-nesting contract. Composite fences (`dma_fence_array`, `dma_fence_chain`) chain children, and `dma_fence_unwrap_for_each` iterates flat. `dma_fence_set_deadline` hints urgency to the producer for power management. Critical for: GPU job tracking, KMS atomic-commit ordering, V4L2/codec pipelines, sync_file/drm_syncobj userspace primitives, dma_resv implicit fencing.

This Tier-3 covers `drivers/dma-buf/dma-fence.c` (~1208 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dma_fence` | per-fence object | `DmaFence` |
| `struct dma_fence_ops` | per-fence vtable | `DmaFenceOps` |
| `struct dma_fence_cb` | per-callback descriptor | `DmaFenceCb` |
| `enum dma_fence_flag_bits` | per-flag enumeration | `DmaFenceFlag` |
| `dma_fence_init()` | per-init (32-bit seqno) | `DmaFence::init` |
| `dma_fence_init64()` | per-init (64-bit seqno) | `DmaFence::init64` |
| `__dma_fence_init()` (static) | per-shared-init helper | `DmaFence::init_inner` |
| `dma_fence_signal()` | per-signal | `DmaFence::signal` |
| `dma_fence_signal_locked()` | per-signal (lock held) | `DmaFence::signal_locked` |
| `dma_fence_signal_timestamp()` | per-signal with ktime | `DmaFence::signal_timestamp` |
| `dma_fence_signal_timestamp_locked()` | per-signal-ts (lock held) | `DmaFence::signal_timestamp_locked` |
| `dma_fence_check_and_signal()` | per-test+signal | `DmaFence::check_and_signal` |
| `dma_fence_check_and_signal_locked()` | per-test+signal (lock held) | `DmaFence::check_and_signal_locked` |
| `dma_fence_wait_timeout()` | per-blocking wait | `DmaFence::wait_timeout` |
| `dma_fence_default_wait()` | per-default impl | `DmaFence::default_wait` |
| `dma_fence_wait_any_timeout()` | per-OR-wait over array | `DmaFence::wait_any_timeout` |
| `dma_fence_add_callback()` | per-add SW callback | `DmaFence::add_callback` |
| `dma_fence_remove_callback()` | per-remove SW callback | `DmaFence::remove_callback` |
| `dma_fence_enable_sw_signaling()` | per-arm SW signaling | `DmaFence::enable_sw_signaling` |
| `__dma_fence_enable_signaling()` (static) | per-arm helper | `DmaFence::enable_signaling_inner` |
| `dma_fence_get_status()` | per-status query | `DmaFence::get_status` |
| `dma_fence_release()` (kref release) | per-final destroy | `DmaFence::release` |
| `dma_fence_free()` | per-default kfree_rcu | `DmaFence::free` |
| `dma_fence_context_alloc()` | per-context ID alloc | `DmaFence::context_alloc` |
| `dma_fence_get_stub()` | per-shared-stub | `DmaFence::stub` |
| `dma_fence_allocate_private_stub()` | per-private-stub | `DmaFence::allocate_private_stub` |
| `dma_fence_set_deadline()` | per-deadline-hint | `DmaFence::set_deadline` |
| `dma_fence_describe()` | per-debug dump | `DmaFence::describe` |
| `dma_fence_driver_name()` | per-RCU driver-name | `DmaFence::driver_name` |
| `dma_fence_timeline_name()` | per-RCU timeline-name | `DmaFence::timeline_name` |
| `dma_fence_begin_signalling()` | per-lockdep-section-start | `DmaFence::begin_signalling` |
| `dma_fence_end_signalling()` | per-lockdep-section-end | `DmaFence::end_signalling` |
| `__dma_fence_might_wait()` | per-lockdep-might-sleep | `DmaFence::might_wait` |
| `dma_fence_context_counter` | per-global context atomic | `DmaFence::CONTEXT_COUNTER` |
| `dma_fence_stub` / `dma_fence_stub_ops` | per-global stub | `DmaFence::STUB` |
| `dma_fence_lockdep_map` | per-lockdep map | `DmaFence::LOCKDEP_MAP` |

## Compatibility contract

REQ-1: struct dma_fence:
- ops: `const struct dma_fence_ops __rcu *` (RCU-protected vtable). Cleared on signal if !release ∧ !wait.
- refcount: `struct kref` (ref-counted lifetime).
- cb_list: callback list_head (drained on signal).
- flags: `unsigned long` bitmap of `DMA_FENCE_FLAG_*`.
- context: `u64` execution-context id (per-engine).
- seqno: `u64` (32-bit unless `DMA_FENCE_FLAG_SEQNO64_BIT` set).
- timestamp: `ktime_t` (valid after `DMA_FENCE_FLAG_TIMESTAMP_BIT`).
- error: `int` (errno or 0).
- rcu: `struct rcu_head` for `kfree_rcu`.
- Inline OR external spinlock: `inline_lock` (when `DMA_FENCE_FLAG_INLINE_LOCK_BIT`) or `extern_lock` (when not).

REQ-2: enum dma_fence_flag_bits:
- DMA_FENCE_FLAG_INITIALIZED_BIT — set by `__dma_fence_init` (only valid bit at pre-signal).
- DMA_FENCE_FLAG_INLINE_LOCK_BIT — fence uses `inline_lock` (set when caller passed lock = NULL).
- DMA_FENCE_FLAG_SEQNO64_BIT — seqno comparison uses signed 64-bit ordering.
- DMA_FENCE_FLAG_SIGNALED_BIT — fence has been signaled (sticky).
- DMA_FENCE_FLAG_TIMESTAMP_BIT — `fence->timestamp` is valid.
- DMA_FENCE_FLAG_ENABLE_SIGNAL_BIT — `enable_signaling` callback has been (or will be) invoked.
- DMA_FENCE_FLAG_USER_BITS — first user bit (driver-defined).

REQ-3: struct dma_fence_ops:
- use_64bit_seqno: bool (deprecated; superseded by `dma_fence_init64`).
- get_driver_name(fence) — required.
- get_timeline_name(fence) — required.
- enable_signaling(fence) — optional; called once. Returns false ⟹ fence is signaled (signaled_locked invoked).
- signaled(fence) — optional poll for HW-only signaled state (used by `dma_fence_is_signaled`).
- wait(fence, intr, timeout) — optional custom wait; deprecated for modules.
- release(fence) — optional custom destroy (default `dma_fence_free`).
- set_deadline(fence, ktime_t) — optional power-management hint.

REQ-4: dma_fence_context_alloc(num):
- WARN if !num.
- Returns `atomic64_fetch_add(num, &dma_fence_context_counter)`. Caller receives a contiguous range `[ret, ret+num)` of unique 64-bit context IDs.

REQ-5: __dma_fence_init(fence, ops, lock, context, seqno, flags):
- BUG_ON(!ops ∨ !ops->get_driver_name ∨ !ops->get_timeline_name).
- kref_init(&fence->refcount).
- RCU_INIT_POINTER(fence->ops, ops).
- INIT_LIST_HEAD(&fence->cb_list).
- fence->context = context; fence->seqno = seqno.
- fence->flags = flags | BIT(DMA_FENCE_FLAG_INITIALIZED_BIT).
- if lock: fence->extern_lock = lock. Else: spin_lock_init(&fence->inline_lock); fence->flags |= BIT(DMA_FENCE_FLAG_INLINE_LOCK_BIT).
- fence->error = 0.
- trace_dma_fence_init.

REQ-6: dma_fence_init(fence, ops, lock, context, seqno):
- __dma_fence_init(fence, ops, lock, context, seqno, 0UL).

REQ-7: dma_fence_init64(fence, ops, lock, context, seqno):
- __dma_fence_init(fence, ops, lock, context, seqno, BIT(DMA_FENCE_FLAG_SEQNO64_BIT)).

REQ-8: dma_fence_signal_timestamp_locked(fence, timestamp):
- dma_fence_assert_held(fence).
- test_and_set_bit(DMA_FENCE_FLAG_SIGNALED_BIT). If was-set: return (sticky / idempotent).
- ops = rcu_dereference_protected(fence->ops, true).
- if !ops->release ∧ !ops->wait: RCU_INIT_POINTER(fence->ops, NULL) (allow ops backing storage to be freed once RCU grace period elapses).
- list_replace(&fence->cb_list, &local_cb_list) — capture pending callbacks.
- fence->timestamp = timestamp; set_bit(TIMESTAMP_BIT).
- trace_dma_fence_signaled.
- For each cb in local_cb_list: INIT_LIST_HEAD(&cb->node); cb->func(fence, cb).

REQ-9: dma_fence_signal_timestamp(fence, timestamp):
- WARN if !fence; return.
- dma_fence_lock_irqsave → signal_timestamp_locked(fence, timestamp) → dma_fence_unlock_irqrestore.

REQ-10: dma_fence_signal_locked(fence):
- signal_timestamp_locked(fence, ktime_get()).

REQ-11: dma_fence_signal(fence):
- WARN if !fence; return.
- tmp = dma_fence_begin_signalling().
- dma_fence_lock_irqsave → signal_timestamp_locked(fence, ktime_get()) → dma_fence_unlock_irqrestore.
- dma_fence_end_signalling(tmp).

REQ-12: dma_fence_check_and_signal_locked(fence):
- ret = dma_fence_test_signaled_flag(fence).
- dma_fence_signal_locked(fence) (idempotent).
- Return ret (true if it was already signaled).

REQ-13: dma_fence_check_and_signal(fence):
- dma_fence_lock_irqsave → check_and_signal_locked → unlock_irqrestore.

REQ-14: dma_fence_wait_timeout(fence, intr, timeout):
- WARN if timeout < 0 ⟹ -EINVAL.
- might_sleep; __dma_fence_might_wait (lockdep priming).
- dma_fence_enable_sw_signaling(fence) (arms enable_signaling if not signaled).
- rcu_read_lock; ops = rcu_dereference(fence->ops); trace_wait_start.
- If ops ∧ ops->wait: rcu_read_unlock; ret = ops->wait(fence, intr, timeout). (Custom wait — deprecated for modules.)
- Else: rcu_read_unlock; ret = dma_fence_default_wait(fence, intr, timeout).
- trace_wait_end.
- Return ret (jiffies remaining, 0 on timeout, -ERESTARTSYS on interrupt, custom errnos possible).

REQ-15: dma_fence_default_wait(fence, intr, timeout):
- ret = timeout ? timeout : 1.
- dma_fence_lock_irqsave.
- If signaled-flag set: goto out.
- If intr ∧ signal_pending(current): ret = -ERESTARTSYS; out.
- If !timeout: ret = 0; out.
- cb.base.func = dma_fence_default_wait_cb; cb.task = current; list_add(&cb.base.node, &fence->cb_list).
- Loop while !signaled ∧ ret > 0:
  - set_current_state(intr ? TASK_INTERRUPTIBLE : TASK_UNINTERRUPTIBLE).
  - dma_fence_unlock_irqrestore.
  - ret = schedule_timeout(ret).
  - dma_fence_lock_irqsave.
  - if (ret > 0 ∧ intr ∧ signal_pending): ret = -ERESTARTSYS.
- If !list_empty(&cb.base.node): list_del.
- __set_current_state(TASK_RUNNING).
- dma_fence_unlock_irqrestore; return ret.

REQ-16: dma_fence_wait_any_timeout(fences, count, intr, timeout, *idx):
- WARN if !fences ∨ !count ∨ timeout < 0 ⟹ -EINVAL.
- If timeout == 0: loop fences; if any signaled set *idx and return 1; else return 0.
- cb = kzalloc array of `default_wait_cb`. None ⟹ -ENOMEM.
- For each fence i: cb[i].task = current; dma_fence_add_callback(fence, &cb[i].base, default_wait_cb). If returned non-zero (already signaled): set *idx = i; goto fence_rm_cb.
- Loop while ret > 0:
  - set_current_state(intr ? INTERRUPTIBLE : UNINTERRUPTIBLE).
  - dma_fence_test_signaled_any(fences, count, idx) ⟹ break.
  - ret = schedule_timeout(ret).
  - if (ret > 0 ∧ intr ∧ signal_pending): ret = -ERESTARTSYS.
- __set_current_state(TASK_RUNNING).
- fence_rm_cb: for each i: dma_fence_remove_callback(fences[i], &cb[i].base).
- err_free_cb: kfree(cb); return ret.

REQ-17: __dma_fence_enable_signaling(fence) (lock held):
- dma_fence_assert_held.
- was_set = test_and_set_bit(ENABLE_SIGNAL_BIT).
- if signaled-flag-set: return false (already signaled).
- ops = rcu_dereference(fence->ops).
- If !was_set ∧ ops ∧ ops->enable_signaling: trace_enable_signal; if !ops->enable_signaling(fence): dma_fence_signal_locked(fence); return false.
- Return true.

REQ-18: dma_fence_enable_sw_signaling(fence):
- dma_fence_lock_irqsave → __dma_fence_enable_signaling → unlock.

REQ-19: dma_fence_add_callback(fence, cb, func):
- WARN if !fence ∨ !func ⟹ -EINVAL.
- If signaled-flag set: INIT_LIST_HEAD(&cb->node); return -ENOENT (caller must NOT expect the callback to fire).
- dma_fence_lock_irqsave.
- If __dma_fence_enable_signaling(fence): cb->func = func; list_add_tail(&cb->node, &fence->cb_list).
- Else: INIT_LIST_HEAD(&cb->node); ret = -ENOENT.
- dma_fence_unlock_irqrestore; return ret.

REQ-20: dma_fence_remove_callback(fence, cb):
- dma_fence_lock_irqsave.
- ret = !list_empty(&cb->node).
- if ret: list_del_init(&cb->node).
- dma_fence_unlock_irqrestore; return ret. (true = removed; false = already fired.)

REQ-21: dma_fence_get_status(fence):
- Wrap dma_fence_get_status_locked under lock.
- Returns 0 (not signaled) | 1 (signaled, no error) | negative errno (signaled with error).

REQ-22: dma_fence_release(kref):
- fence = container_of(kref, struct dma_fence, refcount).
- rcu_read_lock; trace_destroy.
- If !list_empty(&cb_list) ∧ !signaled: WARN("Fence ... released with pending signals!"); set fence->error = -EDEADLK; signal_locked (drains pending cb queue safely).
- ops = rcu_dereference(fence->ops).
- If ops ∧ ops->release: ops->release(fence). Else: dma_fence_free(fence).
- rcu_read_unlock.

REQ-23: dma_fence_free(fence):
- kfree_rcu(fence, rcu).

REQ-24: dma_fence_set_deadline(fence, deadline):
- rcu_read_lock; ops = rcu_dereference(fence->ops).
- If ops ∧ ops->set_deadline ∧ !is_signaled: ops->set_deadline(fence, deadline).
- rcu_read_unlock.

REQ-25: dma_fence_describe(fence, seq):
- rcu_read_lock. Defaults: timeline = "", driver = "", signaled = "".
- If !is_signaled: timeline = dma_fence_timeline_name(fence); driver = dma_fence_driver_name(fence); signaled = "un".
- seq_printf "%llu:%llu %s %s %ssignalled\n" (context, seqno, timeline, driver, signaled).
- rcu_read_unlock.

REQ-26: dma_fence_driver_name(fence) / dma_fence_timeline_name(fence):
- rcu_dereference(fence->ops) → ops->get_{driver,timeline}_name(fence) (if !signaled).
- If signaled: returns "detached-driver" / "signaled-timeline" string literal.
- MUST be called inside an `rcu_read_lock` section.

REQ-27: dma_fence_get_stub():
- dma_fence_get(&dma_fence_stub). The static stub is initialized in `dma_fence_init_stub` (subsys_initcall) with stub ops, ENABLE_SIGNAL_BIT set, then dma_fence_signal()'d immediately. Always SIGNALED, refcount-bumped.

REQ-28: dma_fence_allocate_private_stub(timestamp):
- kzalloc(sizeof(*fence), GFP_KERNEL) — None ⟹ NULL.
- dma_fence_init(fence, &dma_fence_stub_ops, NULL, 0, 0); set ENABLE_SIGNAL_BIT; dma_fence_signal_timestamp(fence, timestamp).

REQ-29: dma_fence_begin_signalling() / dma_fence_end_signalling(cookie):
- CONFIG_LOCKDEP: acquire / release lockdep_map dma_fence_lockdep_map at SUBCLASS-1 (nesting allowed).
- begin: if lock_is_held(map, 1) ⟹ return true (already nested). If in_atomic ⟹ return true (no actual map taken).
- end: cookie==true ⟹ no-op; else lock_release(map).
- !CONFIG_LOCKDEP: both no-op (begin returns true).

REQ-30: __dma_fence_might_wait():
- CONFIG_LOCKDEP: lock_map_acquire(&dma_fence_lockdep_map); lock_map_release. Primes lockdep with the wait/signal hierarchy so cross-driver deadlock conditions are caught at first single-driver test.

REQ-31: Fence lifetime / RCU contract:
- After dma_fence_signal: driver may free the lock and ops backing-storage after an RCU grace period.
- Therefore `fence->ops`, `fence->extern_lock`, `ops->get_driver_name(fence)` strings MUST be touched only inside `rcu_read_lock`.
- `dma_fence_signal_timestamp_locked` clears `fence->ops` to NULL when neither `release` nor `wait` is set (lets module unload while fence still ref'd).

REQ-32: Cross-driver contract (from DOC: fence cross-driver contract):
- Fences must complete in bounded time (driver-defined timeout / GPU hang recovery required).
- Drivers must annotate signaling code with `dma_fence_begin_signalling()` / `_end_signalling()`.
- `dma_fence_wait()` may be called holding `dma_resv_lock()`.
- Code on the signaling path must NOT acquire `dma_resv_lock()`.
- `dma_fence_wait()` allowed in shrinker callbacks ⟹ signaling path cannot allocate GFP_KERNEL.
- `dma_fence_wait()` allowed in mmu_notifier callbacks ⟹ signaling path cannot allocate GFP_NOFS/NOIO; only GFP_ATOMIC.

## Acceptance Criteria

- [ ] AC-1: dma_fence_init initializes refcount = 1, INITIALIZED_BIT set, INLINE_LOCK_BIT set iff lock = NULL.
- [ ] AC-2: dma_fence_init64 additionally sets SEQNO64_BIT.
- [ ] AC-3: dma_fence_context_alloc(N) returns monotonically increasing base, contiguous N ids.
- [ ] AC-4: dma_fence_signal first call sets SIGNALED_BIT and runs all queued callbacks; second call no-ops.
- [ ] AC-5: dma_fence_signal with neither ops->release nor ops->wait set: clears fence->ops to NULL after callbacks drained.
- [ ] AC-6: dma_fence_wait_timeout on already-signaled fence returns `timeout` unchanged (timeout > 0) or 1 (timeout == 0).
- [ ] AC-7: dma_fence_wait_timeout on signaled fence with timeout=0 returns 1.
- [ ] AC-8: dma_fence_wait_timeout with intr=true + pending signal returns -ERESTARTSYS.
- [ ] AC-9: dma_fence_add_callback on already-signaled fence returns -ENOENT and cb->node initialized empty.
- [ ] AC-10: dma_fence_add_callback on unsignaled fence arms enable_signaling (first time only) and queues cb; subsequent signal fires cb->func exactly once.
- [ ] AC-11: dma_fence_remove_callback returns true if cb still queued, false if already fired.
- [ ] AC-12: dma_fence_release on fence with non-empty cb_list and !signaled WARNs and force-signals with error = -EDEADLK.
- [ ] AC-13: dma_fence_release calls ops->release if set, else dma_fence_free (kfree_rcu).
- [ ] AC-14: dma_fence_set_deadline: signaled fence ⟹ no-op; unsignaled fence ⟹ ops->set_deadline if set.
- [ ] AC-15: dma_fence_get_stub returns a fence with SIGNALED_BIT set and refcount bumped.
- [ ] AC-16: dma_fence_wait_any_timeout returns index of first signaled fence and remaining jiffies.
- [ ] AC-17: dma_fence_begin_signalling/_end_signalling annotations prime lockdep (CONFIG_LOCKDEP) without affecting runtime semantics.

## Architecture

```
struct DmaFence {
  ops: RcuPtr<DmaFenceOps>,             // RCU-protected vtable
  refcount: Kref,
  cb_list: ListHead<DmaFenceCb>,
  flags: AtomicUsize,                   // BIT(DmaFenceFlag::*)
  context: u64,
  seqno: u64,
  timestamp: KTime,
  error: i32,
  rcu: RcuHead,
  // Discriminated by INLINE_LOCK_BIT:
  inline_lock: SpinLock,                // when set
  extern_lock: Option<&'static SpinLock>, // when not set
}

#[repr(usize)]
enum DmaFenceFlag {
  Initialized   = 0,
  InlineLock    = 1,
  Seqno64       = 2,
  Signaled      = 3,
  Timestamp     = 4,
  EnableSignal  = 5,
  UserBits      = 6,
}

struct DmaFenceCb {
  node: ListNode,
  func: fn(&DmaFence, &mut DmaFenceCb),
}

struct DmaFenceOps {
  use_64bit_seqno: bool,                // deprecated; use init64
  get_driver_name: fn(&DmaFence) -> &str,        // required
  get_timeline_name: fn(&DmaFence) -> &str,      // required
  enable_signaling: Option<fn(&DmaFence) -> bool>,
  signaled: Option<fn(&DmaFence) -> bool>,
  wait: Option<fn(&DmaFence, intr: bool, timeout: i64) -> i64>,
  release: Option<fn(&DmaFence)>,
  set_deadline: Option<fn(&DmaFence, KTime)>,
}
```

`DmaFence::init(fence, ops, lock, context, seqno)`:
1. BUG_ON(ops.is_none() ∨ !ops.get_driver_name ∨ !ops.get_timeline_name).
2. kref_init(&fence.refcount).
3. RCU_INIT_POINTER(fence.ops, ops).
4. INIT_LIST_HEAD(&fence.cb_list).
5. fence.context = context; fence.seqno = seqno.
6. fence.flags = BIT(Initialized).
7. If lock.is_some(): fence.extern_lock = lock; else: spin_lock_init(&fence.inline_lock); fence.flags |= BIT(InlineLock).
8. fence.error = 0.
9. trace_dma_fence_init(fence).

`DmaFence::init64(fence, ops, lock, context, seqno)`:
1. As `init`, plus fence.flags |= BIT(Seqno64) post-step 6.

`DmaFence::signal(fence)`:
1. WARN if fence is None; return.
2. cookie = DmaFence::begin_signalling().
3. flags = dma_fence_lock_irqsave(fence).
4. DmaFence::signal_timestamp_locked(fence, ktime_get()).
5. dma_fence_unlock_irqrestore(fence, flags).
6. DmaFence::end_signalling(cookie).

`DmaFence::signal_timestamp_locked(fence, timestamp)`:
1. dma_fence_assert_held(fence).
2. if test_and_set_bit(Signaled, &fence.flags): return (idempotent).
3. ops = rcu_dereference_protected(fence.ops, true).
4. if !ops.release ∧ !ops.wait: RCU_INIT_POINTER(fence.ops, None) (let module unload after RCU grace).
5. list_replace(&fence.cb_list, &local).
6. fence.timestamp = timestamp; set_bit(Timestamp).
7. trace_dma_fence_signaled(fence).
8. For cb in local: INIT_LIST_HEAD(&cb.node); cb.func(fence, &mut cb).

`DmaFence::wait_timeout(fence, intr, timeout) -> i64`:
1. WARN if timeout < 0 ⟹ -EINVAL.
2. might_sleep; DmaFence::might_wait() (lockdep).
3. DmaFence::enable_sw_signaling(fence).
4. rcu_read_lock; ops = rcu_dereference(fence.ops); trace_wait_start.
5. if ops ∧ ops.wait: rcu_read_unlock; ret = ops.wait(fence, intr, timeout). Else: rcu_read_unlock; ret = DmaFence::default_wait(fence, intr, timeout).
6. trace_wait_end; return ret.

`DmaFence::default_wait(fence, intr, timeout) -> i64`:
1. ret = if timeout != 0 { timeout } else { 1 }.
2. flags = dma_fence_lock_irqsave(fence).
3. If signaled: goto out.
4. If intr ∧ signal_pending(current): ret = -ERESTARTSYS; goto out.
5. If !timeout: ret = 0; goto out.
6. cb.base.func = default_wait_cb; cb.task = current; list_add(&cb.base.node, &fence.cb_list).
7. While !signaled ∧ ret > 0:
   - set_current_state(intr ? Interruptible : Uninterruptible).
   - dma_fence_unlock_irqrestore(fence, flags).
   - ret = schedule_timeout(ret).
   - flags = dma_fence_lock_irqsave(fence).
   - If ret > 0 ∧ intr ∧ signal_pending: ret = -ERESTARTSYS.
8. If !cb.base.node.is_empty(): list_del.
9. __set_current_state(Running).
10. out: dma_fence_unlock_irqrestore(fence, flags); return ret.

`DmaFence::add_callback(fence, cb, func) -> Errno<()>`:
1. WARN if !fence ∨ !func ⟹ Err(EINVAL).
2. If signaled-flag set: INIT_LIST_HEAD(&cb.node); Err(ENOENT).
3. flags = dma_fence_lock_irqsave.
4. If __dma_fence_enable_signaling(fence): cb.func = func; list_add_tail(&cb.node, &fence.cb_list); ret = Ok.
5. Else: INIT_LIST_HEAD(&cb.node); ret = Err(ENOENT).
6. dma_fence_unlock_irqrestore; return ret.

`DmaFence::release(kref)`:
1. fence = container_of(kref, DmaFence, refcount).
2. rcu_read_lock; trace_destroy.
3. If !fence.cb_list.is_empty() ∧ !signaled:
   - WARN "released with pending signals".
   - dma_fence_lock_irqsave; fence.error = -EDEADLK; signal_locked; unlock_irqrestore.
4. ops = rcu_dereference(fence.ops).
5. if ops ∧ ops.release: ops.release(fence). Else: DmaFence::free(fence).
6. rcu_read_unlock.

`DmaFence::free(fence)`:
1. kfree_rcu(fence, rcu).

`DmaFence::context_alloc(num) -> u64`:
1. WARN if !num.
2. atomic64_fetch_add(num, &CONTEXT_COUNTER).

`DmaFence::set_deadline(fence, deadline)`:
1. rcu_read_lock; ops = rcu_dereference(fence.ops).
2. if ops ∧ ops.set_deadline ∧ !is_signaled: ops.set_deadline(fence, deadline).
3. rcu_read_unlock.

`DmaFence::stub() -> &'static DmaFence`:
1. Returns dma_fence_get(&STUB) (always signaled).

`DmaFence::wait_any_timeout(fences, count, intr, timeout, *idx) -> i64`:
1. WARN if !fences ∨ !count ∨ timeout < 0 ⟹ -EINVAL.
2. If timeout == 0: loop; first signaled fence → *idx = i; return 1. Else 0.
3. cb = kzalloc(count * sizeof(default_wait_cb)).
4. For i in 0..count: cb[i].task = current; if DmaFence::add_callback(fences[i], &cb[i].base, default_wait_cb).is_err(): // already signaled
   - if idx: *idx = i; goto fence_rm_cb.
5. Loop while ret > 0:
   - set_current_state.
   - test_signaled_any(fences, count, idx) ⟹ break.
   - ret = schedule_timeout(ret).
   - intr/signal: ret = -ERESTARTSYS.
6. fence_rm_cb: for each fence i: DmaFence::remove_callback(fences[i], &cb[i].base).
7. kfree(cb); return ret.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `signal_idempotent` | INVARIANT | per-signal: SIGNALED-bit is sticky; second call no-ops. |
| `signal_drains_cb_list` | INVARIANT | per-signal: every cb in cb_list at signal time is invoked exactly once. |
| `signal_ops_cleared_when_safe` | INVARIANT | per-signal: !ops.release ∧ !ops.wait ⟹ ops cleared. |
| `add_callback_signaled_returns_enoent` | INVARIANT | per-add_callback: signaled fence ⟹ ENOENT, no list insert. |
| `enable_signaling_once` | INVARIANT | per-add_callback / per-enable_sw_signaling: ops.enable_signaling called at most once. |
| `release_warns_pending_cb` | INVARIANT | per-release: !signaled ∧ cb_list non-empty ⟹ WARN + force-signal. |
| `wait_timeout_negative` | INVARIANT | per-wait_timeout: timeout < 0 ⟹ -EINVAL. |
| `wait_signal_pending` | INVARIANT | per-default_wait: intr ∧ signal_pending ⟹ -ERESTARTSYS. |
| `context_alloc_monotonic` | INVARIANT | per-context_alloc(N): returns x, next call returns ≥ x + N. |
| `inline_vs_extern_lock_consistent` | INVARIANT | per-init: lock=None ⟹ INLINE_LOCK_BIT set; lock=Some ⟹ extern_lock set, bit clear. |
| `rcu_ops_access_rcu_locked` | INVARIANT | per-driver_name / per-timeline_name / per-set_deadline / per-wait_timeout: rcu_read_lock held during ops deref. |
| `stub_always_signaled` | INVARIANT | dma_fence_get_stub returns fence with SIGNALED bit. |

### Layer 2: TLA+

`drivers/dma-buf/dma-fence.tla`:
- Per-init + per-add_callback + per-enable_signaling + per-signal + per-wait + per-release lifecycle.
- Properties:
  - `safety_signal_once` — Signal transition fires at most once.
  - `safety_cb_fires_once_or_never` — cb queued before signal ⟹ fires once; cb queued after signal ⟹ never fires (add_callback returns ENOENT).
  - `safety_seqno_monotonic_in_context` — within fixed context, seqno on signaled fences is monotonic per producer-discipline assumption.
  - `safety_refcount_balanced` — get / put balanced over lifetime.
  - `safety_no_use_after_signal_without_rcu` — ops access without rcu_read_lock ⟹ violation.
  - `liveness_signal_eventually_wakes_waiters` — signal ⟹ all default_wait_cb invocations occur.
  - `liveness_wait_terminates` — wait_timeout terminates with signaled / 0 / -ERESTARTSYS / negative-errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DmaFence::init` post: refcount = 1 ∧ INITIALIZED set ∧ INLINE_LOCK iff lock==None | `DmaFence::init` |
| `DmaFence::init64` post: SEQNO64 set | `DmaFence::init64` |
| `DmaFence::signal_timestamp_locked` post: SIGNALED ∧ TIMESTAMP set ∧ cb_list drained | `DmaFence::signal_timestamp_locked` |
| `DmaFence::wait_timeout` post: ret ∈ {signed-jiffies, 0, -ERESTARTSYS, custom} | `DmaFence::wait_timeout` |
| `DmaFence::add_callback` post: !signaled ⟹ cb in cb_list ∧ ENABLE_SIGNAL set | `DmaFence::add_callback` |
| `DmaFence::release` pre: !signaled ⟹ error = -EDEADLK ∧ signal forced | `DmaFence::release` |
| `DmaFence::context_alloc` post: ret unique across all callers | `DmaFence::context_alloc` |
| `DmaFence::set_deadline` post: signaled ⟹ no-op; unsignaled ⟹ ops.set_deadline if defined | `DmaFence::set_deadline` |
| `DmaFence::stub` post: returned fence SIGNALED ∧ refcount bumped | `DmaFence::stub` |
| `DmaFence::driver_name` precond: rcu_read_lock held | `DmaFence::driver_name` |

### Layer 4: Verus/Creusot functional

`Per-init → (refcount=1, INITIALIZED, lock chosen) → per-add_callback (enable_signaling armed first-time, cb queued) → per-signal (SIGNALED sticky, cb_list drained, ops cleared if standalone, timestamp recorded) → per-wait_timeout (custom or default; lockdep primed via __dma_fence_might_wait) → per-put → per-release (force-signal+warn on leaked cb; ops.release else kfree_rcu)` semantic equivalence with `Documentation/driver-api/dma-buf.rst` § fences, `include/linux/dma-fence.h` contracts, and the cross-driver-contract rules (bounded completion time, no resv_lock on signaling path, GFP_ATOMIC only for signaling-path allocation under mmu_notifier).

Composite fence integration (dma_fence_array, dma_fence_chain, dma_fence_unwrap_for_each) treated in their dedicated Tier-3 designs; the primitive contract here is what those composites layer on.

## Hardening

(Inherits row-1 features from `drivers/dma-buf/00-overview.md` § Hardening.)

DMA-fence reinforcement:

- **Per-RCU-protected ops** — defense against per-driver-module-unload while waiter still holds fence ref.
- **Per-ops cleared on signal when standalone** — defense against per-driver-data-freed UAF.
- **Per-INLINE_LOCK_BIT vs extern_lock discipline** — defense against per-uninitialized-lock UAF in long-lived fences.
- **Per-test_and_set_bit(SIGNALED) sticky** — defense against per-double-signal corrupting cb_list.
- **Per-cb_list drained under lock + list_replace** — defense against per-concurrent-add during signal dispatch.
- **Per-add_callback-after-signal returns ENOENT** — defense against per-lost-callback (caller forced to check or check_and_signal).
- **Per-release WARN+force-signal pending-cb** — defense against per-refcounting-bug leaving callbacks orphaned.
- **Per-might_sleep in wait_timeout** — defense against per-atomic-wait deadlock.
- **Per-lockdep priming (begin/end_signalling + might_wait)** — defense against per-cross-driver deadlock in dma_resv_lock vs dma_fence_signal nesting.
- **Per-default_wait correct intr / TASK_RUNNING restoration** — defense against per-stuck-task-state if cancelled mid-wait.
- **Per-wait_any cleanup of all add_callbacks on error** — defense against per-dangling-cb on OOM/signal/path.
- **Per-context-alloc 64-bit atomic** — defense against per-id-overlap collision under heavy GPU workload.
- **Per-stub fence always-signaled + ref-counted** — defense against per-NULL-fence import path (sync_file_export with no implicit fences).
- **Per-set_deadline gated by !signaled + rcu** — defense against per-signaled-fence ops UAF in PM-hint path.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on `sync_file` IOCTL payload + fence export/import buffers.
- **PAX_KERNEXEC** — W^X enforcement on fence callback (`cb_func`) dispatch.
- **PAX_RANDKSTACK** — kernel-stack randomization on `dma_fence_wait` syscall entry.
- **PAX_REFCOUNT** — saturating `dma_fence->refcount` (kref) and per-context atomic64.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `dma_fence` slabs and `dma_fence_cb` callback closures.
- **PAX_UDEREF** — SMAP/SMEP enforcement on sync_file IOCTL user-pointer access.
- **PAX_RAP / kCFI** — `dma_fence_ops` (`get_driver_name` / `get_timeline_name` / `enable_signaling` / `signaled` / `wait` / `release`) and `dma_fence_cb.func` indirect calls hardened; per-exporter ops `static const`.
- **GRKERNSEC_HIDESYM** — kernel-pointer hiding in fence debug output (timeline name + context ID OK; pointers suppressed).
- **GRKERNSEC_DMESG** — syslog restriction on fence WARN (refcount underflow, pending-cb on release).
- **PAX_CONSTIFY_PLUGIN** — every static `dma_fence_ops` literal `static const`.
- **CAP_SYS_ADMIN** for any debugfs fence-state mutation.
- **PAX_SIZE_OVERFLOW** — `seqno`, `context`, timeout arithmetic checked.
- **GRKERNSEC_SYSCTL** — fence-related sysctl (lockdep priming, timeout caps) locked at boot.
- **LSM `security_file_ioctl`** — sync_file IOCTL gated per GR-RBAC subject.

Per-doc rationale: dma-fence callbacks are arbitrary indirect calls scheduled into a callback chain that fires under spinlock from any context (IRQ, NMI-soft, RT thread); a hijacked `cb_func` becomes the easiest path to kernel-mode RCE in GPU stacks. PAX_RAP locks both `dma_fence_ops` and the per-callback `cb_func` indirect; PAX_REFCOUNT prevents UAF on the fence under RCU + signal races, PAX_MEMORY_SANITIZE wipes callback closures (which carry driver-private data), and PAX_KERNEXEC ensures the callback target page is non-writable kernel text.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/dma-buf/dma-fence-array.c` OR-array composite (separate Tier-3)
- `drivers/dma-buf/dma-fence-chain.c` timeline-chain composite (separate Tier-3)
- `drivers/dma-buf/dma-fence-unwrap.c` flattening iterator (separate Tier-3)
- `drivers/dma-buf/sync_file.c` userspace sync_file (separate Tier-3)
- `drivers/dma-buf/sw_sync.c` software fence timeline (separate Tier-3)
- `drivers/dma-buf/dma-resv.c` reservation object (separate Tier-3)
- `drivers/dma-buf/dma-buf.c` buffer object (Tier-3 dma-buf.md)
- DRM scheduler / amdgpu / i915 producer integrations
- Hardware-fence signaling (interrupt + workqueue path inside individual GPU drivers)
- Implementation code
