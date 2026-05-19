# Tier-3: kernel/locking/lockdep.c — Runtime lock-dependency validator

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/locking/00-overview.md
upstream-paths:
  - kernel/locking/lockdep.c (~6880 lines)
  - kernel/locking/lockdep_internals.h
  - include/linux/lockdep.h, include/linux/lockdep_types.h
  - Documentation/locking/lockdep-design.rst
-->

## Summary

Lockdep is the runtime lock-dependency validator. Per-lock instance the kernel registers a `struct lockdep_map` (name, key pointer, per-subclass class-cache, wait-type, lock-type) carried inside every lock primitive (spinlock_t, rwlock_t, mutex, rwsem, completion, ...). Per-lock-acquisition site `lock_acquire(map, subclass, trylock, read, check, nest_lock, ip)` resolves the map to a `struct lock_class` (interning by `struct lock_class_key`), pushes a `struct held_lock` onto `current->held_locks[]`, then runs the four kernel deadlock checks: (i) `check_deadlock` — same-class recursion (with rwlock read-after-read exemption); (ii) `check_noncircular` — BFS forward from `next` to `prev` over the class dependency graph to forbid cycles A→B and B→A; (iii) `check_irq_usage` — BFS to forbid irq-safe vs irq-unsafe mixing across the same chain; (iv) `check_wait_context` — wait-type ordering (raw < spin < sleeping). Per-graph the validator caches each successfully-validated chain by chain-hash so each unique acquisition path is verified only once. Per-key registration `lockdep_register_key` adds a dynamic key (e.g. for kmem_cache locks) so dynamically-allocated locks are accepted; `lockdep_unregister_key` retires it (RCU-deferred zap of classes that pointed at it). Per-IRQ-state tracking `lockdep_hardirqs_on / off` and `lockdep_softirqs_on / off` mark the irq context inside which subsequent locks are acquired, so the validator can classify a class as `HARDIRQ_USED_*` / `SOFTIRQ_USED_*` / `ENABLED_HARDIRQ` / `ENABLED_SOFTIRQ`. Per-cycle / per-inversion the validator prints a structured `WARNING: possible circular locking dependency detected` or `WARNING: possible irq lock inversion dependency` report, then sets `debug_locks = 0` to disable itself (one-shot). Critical for: catching AB-BA deadlocks pre-deployment, locking-rule enforcement (irq-safe vs sleep), recursive-lock bug detection.

This Tier-3 covers `kernel/locking/lockdep.c` (~6880 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct lockdep_map` | per-lock-instance handle (in lock primitives) | `LockdepMap` |
| `struct lock_class_key` | per-class identity token | `LockClassKey` |
| `struct lock_class` | per-class node in dep graph | `LockClass` |
| `struct held_lock` | per-acquisition stack entry | `HeldLock` |
| `struct lock_list` | per-class-edge in dep graph | `LockListEntry` |
| `struct lock_chain` | per-chain-hash cached chain | `LockChain` |
| `lock_classes[MAX_LOCKDEP_KEYS]` | per-graph class pool | `Lockdep::CLASSES` |
| `list_entries[MAX_LOCKDEP_ENTRIES]` | per-graph edge pool | `Lockdep::LIST_ENTRIES` |
| `lock_chains[MAX_LOCKDEP_CHAINS]` | per-graph chain cache | `Lockdep::CHAINS` |
| `chain_hlocks[]` | per-chain `hlock_id` storage | `Lockdep::CHAIN_HLOCKS` |
| `classhash_table[]` / `chainhash_table[]` | per-hash buckets | `Lockdep::CLASSHASH` / `CHAINHASH` |
| `lock_keys_hash[]` | per-dynamic-key buckets | `Lockdep::KEYS_HASH` |
| `lockdep_init()` | per-boot announce | `Lockdep::init` |
| `lockdep_init_map_type()` | per-instance attach | `Lockdep::init_map_type` |
| `lockdep_register_key()` | per-dynamic-key register | `Lockdep::register_key` |
| `lockdep_unregister_key()` | per-dynamic-key retire | `Lockdep::unregister_key` |
| `register_lock_class()` | per-instance intern to class | `Lockdep::register_lock_class` |
| `look_up_lock_class()` | per-cache lookup | `Lockdep::look_up_lock_class` |
| `assign_lock_key()` | per-static-obj key from address | `Lockdep::assign_lock_key` |
| `lock_acquire()` | per-acquire hook | `Lockdep::lock_acquire` |
| `lock_release()` | per-release hook | `Lockdep::lock_release` |
| `lock_sync()` | per-RCU-style sync annotate | `Lockdep::lock_sync` |
| `lock_acquired()` / `lock_contended()` | per-lock-stat | `Lockdep::lock_acquired` / `lock_contended` |
| `__lock_acquire()` | per-acquire core | `Lockdep::lock_acquire_inner` |
| `__lock_release()` | per-release core | `Lockdep::lock_release_inner` |
| `check_deadlock()` | per-same-class recursion | `Lockdep::check_deadlock` |
| `check_noncircular()` | per-cycle BFS | `Lockdep::check_noncircular` |
| `check_prev_add()` | per-edge add+validate | `Lockdep::check_prev_add` |
| `check_prevs_add()` | per-acquire trylock-chain walk | `Lockdep::check_prevs_add` |
| `check_irq_usage()` | per-irq inversion | `Lockdep::check_irq_usage` |
| `check_wait_context()` | per-wait-type ordering | `Lockdep::check_wait_context` |
| `validate_chain()` | per-chain-hash validate-once | `Lockdep::validate_chain` |
| `mark_lock()` / `mark_usage()` | per-usage-bit set | `Lockdep::mark_lock` / `mark_usage` |
| `mark_held_locks()` | per-irq-on/off propagate | `Lockdep::mark_held_locks` |
| `lockdep_hardirqs_on/off()` / `_prepare()` | per-IRQ-state | `Lockdep::hardirqs_on/off` |
| `lockdep_softirqs_on/off()` | per-softirq-state | `Lockdep::softirqs_on/off` |
| `lock_is_held_type()` | per-assert held | `Lockdep::lock_is_held_type` |
| `lock_pin_lock` / `lock_unpin_lock` | per-pin cookie | `Lockdep::lock_pin` / `unpin` |
| `print_circular_bug()` | per-cycle report | `Lockdep::print_circular_bug` |
| `print_bad_irq_dependency()` | per-irq-inversion report | `Lockdep::print_bad_irq_dependency` |
| `print_deadlock_bug()` | per-recursion report | `Lockdep::print_deadlock_bug` |
| `debug_check_no_locks_freed()` | per-kfree held-lock check | `Lockdep::debug_check_no_locks_freed` |
| `debug_check_no_locks_held()` | per-exit held-lock check | `Lockdep::debug_check_no_locks_held` |
| `lockdep_free_key_range()` | per-module-unload zap | `Lockdep::free_key_range` |
| `lockdep_reset_lock()` | per-instance zap | `Lockdep::reset_lock` |
| `lockdep_sys_exit()` | per-syscall return guard | `Lockdep::sys_exit` |
| `lockdep_rcu_suspicious()` | per-RCU lockdep splat | `Lockdep::rcu_suspicious` |

## Compatibility contract

REQ-1: struct lockdep_map layout (embedded in spinlock_t / mutex / rwsem / rwlock_t / completion / ...):
- key: *const LockClassKey (the class identity).
- class_cache: [Option<&LockClass>; NR_LOCKDEP_CACHING_CLASSES] (cached per-subclass).
- name: &'static str (kernel symbol name from the LOCK_INIT macro).
- wait_type_outer: u8 (LD_WAIT_*).
- wait_type_inner: u8.
- lock_type: u8 (LD_LOCK_NORMAL / _PERCPU / _WAIT_OVERRIDE).
- cpu: i32 (lockstat).
- ip: u64 (lockstat).

REQ-2: struct lock_class_key:
- subkeys: [LockdepSubclassKey; MAX_LOCKDEP_SUBCLASSES] (one identity per subclass).
- hash_entry: HlistNode (if dynamic, lives on lock_keys_hash bucket).
- Static keys live in `.data` / `.bss` and pass `static_obj()`; dynamic keys are explicitly registered.

REQ-3: struct lock_class:
- key: *const LockdepSubclassKey.
- name: &'static str.
- subclass: u8.
- name_version: i32 (disambiguates same-name classes).
- locks_before: list_head — edges to classes ever held before this class.
- locks_after: list_head — edges to classes ever held after this class.
- usage_mask: u64 — bitmap over LOCK_USED_IN_HARDIRQ, _IN_SOFTIRQ, _ENABLED_HARDIRQ, _ENABLED_SOFTIRQ + READ variants + LOCK_USED.
- usage_traces: [LockTrace*; LOCK_TRACE_STATES].
- hash_entry: HlistNode (classhash_table bucket).
- lock_entry: ListHead (member of free_lock_classes or all_lock_classes).
- ops: u64 (statistics).
- wait_type_inner / wait_type_outer / lock_type.
- cmp_fn: Option<lock_cmp_fn> (per-class custom comparator for nested-lock ordering).
- print_fn.

REQ-4: struct held_lock (one per acquired lock per task):
- prev_chain_key: u64.
- acquire_ip: u64.
- instance: *const LockdepMap.
- nest_lock: *const LockdepMap (if rwsem nesting).
- waittime_stamp / holdtime_stamp: u64 (lockstat).
- class_idx: u32 (index into lock_classes[]).
- irq_context: u8 (0 = task, 1 = softirq, 2 = hardirq).
- trylock: bit.
- read: 2-bit (0 = write, 1 = recursive read, 2 = non-recursive read).
- check: bit (enable validation).
- hardirqs_off: bit.
- references: u32 (nested same-class same-instance count).
- pin_count: u32.
- sync: bit (lock_sync annotation).

REQ-5: lockdep_init_map_type(lock, name, key, subclass, inner, outer, lock_type):
- For i in 0..NR_LOCKDEP_CACHING_CLASSES: lock.class_cache[i] = NULL.
- if LOCK_STAT: lock.cpu = raw_smp_processor_id().
- if !name: lock.name = "NULL"; return.
- lock.name = name; lock.wait_type_outer = outer; lock.wait_type_inner = inner; lock.lock_type = lock_type.
- if !key: return.
- if !static_obj(key) ∧ !is_dynamic_key(key): pr_err("BUG: key %px has not been registered!"); WARN_ON; return.
- lock.key = key.
- if subclass: register_lock_class(lock, subclass, force=1) under irqsave / lockdep_recursion guard.

REQ-6: register_lock_class(lock, subclass, force):
- DEBUG_LOCKS_WARN_ON(!irqs_disabled).
- class = look_up_lock_class(lock, subclass).
- if class: goto out_set_class_cache.
- if !lock.key: assign_lock_key(lock).
- else if !static_obj(lock.key) ∧ !is_dynamic_key(lock.key): return NULL.
- key = lock.key.subkeys + subclass.
- hash_head = classhashentry(key).
- if !graph_lock: return NULL.
- /* Recheck under graph_lock */
- hlist_for_each_entry_rcu(class, hash_head, hash_entry): if class.key == key: goto out_unlock_set.
- init_data_structures_once().
- class = list_first_entry_or_null(free_lock_classes).
- if !class: pr_err("BUG: MAX_LOCKDEP_KEYS too low!"); dump_stack; return NULL.
- nr_lock_classes++; set lock_classes_in_use bit; debug_atomic_inc(nr_unused_locks).
- class.key = key; class.name = lock.name; class.subclass = subclass.
- class.name_version = count_matching_names(class).
- class.wait_type_inner = lock.wait_type_inner; class.wait_type_outer = lock.wait_type_outer; class.lock_type = lock.lock_type.
- hlist_add_head_rcu(class.hash_entry, hash_head).
- list_move_tail(class.lock_entry, all_lock_classes).
- if verbose(class): print "new class %px: %s"; dump_stack.
- out_unlock_set: graph_unlock.
- out_set_class_cache: cache class into lock.class_cache[subclass] (or [0] if subclass==0 or force).

REQ-7: lockdep_register_key(key):
- WARN_ON_ONCE(static_obj(key)) — static keys must not be registered.
- raw_local_irq_save; graph_lock.
- hash_head = keyhashentry(key).
- For k in hash_head: WARN_ON_ONCE(k == key) — double-register.
- hlist_add_head_rcu(key.hash_entry, hash_head).
- nr_dynamic_keys++.
- graph_unlock; raw_local_irq_restore.

REQ-8: lock_acquire(lock, subclass, trylock, read, check, nest_lock, ip):
- trace_lock_acquire(...).
- if !debug_locks: return.
- kasan_check_byte(lock).
- if !lockdep_enabled(): /* recursion / not-yet-init / disabled */ if lockdep_nmi ∧ !trylock: verify_lock_unused; return.
- raw_local_irq_save; check_flags(flags).
- lockdep_recursion_inc.
- __lock_acquire(lock, subclass, trylock, read, check, irqs_disabled_flags(flags), nest_lock, ip, references=0, pin_count=0, sync=0).
- lockdep_recursion_finish; raw_local_irq_restore.

REQ-9: __lock_acquire core:
- curr = current.
- if !debug_locks ∨ lock.key == &__lockdep_no_track__: return 0.
- if !prove_locking ∨ lock.key == &__lockdep_no_validate__: check = 0.
- if subclass >= MAX_LOCKDEP_SUBCLASSES: WARN; return 0.
- class = lock.class_cache[subclass] (if subclass < NR_LOCKDEP_CACHING_CLASSES) else NULL.
- if !class: class = register_lock_class(lock, subclass, 0).
- if !class: return 0.
- depth = curr.lockdep_depth.
- if depth >= MAX_LOCK_DEPTH: WARN; return 0.
- class_idx = class - lock_classes.
- /* References merge for nested same-class same-nest */
- if depth ∧ !sync:
  - hlock = curr.held_locks + depth - 1.
  - if hlock.class_idx == class_idx ∧ nest_lock: hlock.references += references+1; return 2.
- hlock = curr.held_locks + depth.
- Fill hlock: class_idx, acquire_ip, instance, nest_lock, irq_context = task_irq_context(curr), trylock, read, check, sync, hardirqs_off, references, pin_count.
- if LOCK_STAT: waittime_stamp = 0; holdtime_stamp = lockstat_clock().
- if check_wait_context(curr, hlock) fails: return 0.
- if !mark_usage(curr, hlock, check): return 0.
- /* Chain key */
- chain_key = curr.curr_chain_key.
- if !depth: WARN_ON(chain_key ≠ INITIAL_CHAIN_KEY); chain_head = 1.
- hlock.prev_chain_key = chain_key.
- if separate_irq_context(curr, hlock): chain_key = INITIAL_CHAIN_KEY; chain_head = 1.
- chain_key = iterate_chain_key(chain_key, hlock_id(hlock)).
- if nest_lock ∧ !__lock_is_held(nest_lock, -1): print_lock_nested_lock_not_held; return 0.
- if !validate_chain(curr, hlock, chain_head, chain_key): return 0.
- if sync: return 1 (no critical section).
- curr.curr_chain_key = chain_key; curr.lockdep_depth++.
- check_chain_key(curr).
- if curr.lockdep_depth >= MAX_LOCK_DEPTH: debug_locks_off; print_lockdep_off; return 0.
- return 1.

REQ-10: validate_chain(curr, hlock, chain_head, chain_key):
- /* Cache: was this chain already verified? */
- if lookup_chain_cache_add(chain_key, ...): if duplicate-collision: print_collision; return 0.
- /* Run checks unique to first encounter */
- if check_deadlock(curr, hlock) returns 0: bail.
- if curr.lockdep_depth > 0 ∧ check: check_prevs_add(curr, hlock).

REQ-11: check_deadlock(curr, next):
- For i in 0..curr.lockdep_depth:
  - prev = curr.held_locks + i.
  - if prev.instance == next.nest_lock: nest = prev.
  - if hlock_class(prev) ≠ hlock_class(next): continue.
  - /* read-after-read recursion exempt: next.read==2 ∧ prev.read */
  - if next.read == 2 ∧ prev.read: continue.
  - class = hlock_class(prev).
  - if class.cmp_fn ∧ cmp_fn(prev.instance, next.instance) < 0: continue.
  - if nest: return 2 (nest_lock serialises).
  - print_deadlock_bug(curr, prev, next); return 0.
- return 1.

REQ-12: check_prev_add(curr, prev, next, distance, trace):
- if !hlock_class(prev).key ∨ !hlock_class(next).key: WARN("Detected use-after-free of lock class"); return 2.
- if prev.class_idx == next.class_idx ∧ class.cmp_fn(prev, next) < 0: return 2.
- /* BFS forward: does next → ... → prev exist? */
- ret = check_noncircular(next, prev, trace).
- if bfs_error(ret) ∨ ret == BFS_RMATCH: print_circular_bug; return 0.
- if !check_irq_usage(curr, prev, next): return 0.
- /* Dup-edge: already prev → next ? */
- For entry in hlock_class(prev).locks_after:
  - if entry.class == hlock_class(next): update distance / dep flags; check reverse edge in next.locks_before; return.
- /* Redundant via existing forward dep? */
- ret = check_redundant(prev, next).
- if bfs_error(ret): return 0.
- else if ret == BFS_RMATCH: return 2.
- if !trace: trace = save_trace().
- add_lock_to_list(next.class, prev.class, &prev.class.locks_after, distance, calc_dep(prev, next), trace).
- add_lock_to_list(prev.class, next.class, &next.class.locks_before, distance, calc_depb(prev, next), trace).
- return 2.

REQ-13: check_noncircular(src, target, trace):
- BFS forwards from src over class.locks_after using `hlock_conflict` as match predicate.
- Bounded by MAX_CIRCULAR_QUEUE_SIZE; on overflow returns BFS_EQUEUEFULL.
- BFS_RMATCH ⟹ cycle (target reachable from src).

REQ-14: check_irq_usage(curr, prev, next):
- /* Step 1: backward BFS from prev, accumulate usage_mask */
- bfs_init_rootb(&this, prev).
- __bfs_backwards(&this, &usage_mask, usage_accumulate, usage_skip).
- usage_mask &= LOCKF_USED_IN_IRQ_ALL.
- if !usage_mask: return 1 (no irq-safe predecessor).
- /* Step 2: forward BFS from next; mask = exclusive_mask(usage_mask) */
- forward_mask = exclusive_mask(usage_mask).
- bfs_init_root(&that, next); find_usage_forwards(&that, forward_mask, &target_entry1).
- if BFS_RNOMATCH: return 1.
- /* Step 3: backward find matching backward witness */
- backward_mask = original_mask(target_entry1.class.usage_mask & LOCKF_ENABLED_IRQ_ALL).
- find_usage_backwards(&this, backward_mask, &target_entry).
- /* Step 4: narrow to pair, report */
- find_exclusive_match(target_entry.class.usage_mask, target_entry1.class.usage_mask, &backward_bit, &forward_bit).
- print_bad_irq_dependency(curr, &this, &that, target_entry, target_entry1, prev, next, backward_bit, forward_bit, state_name(backward_bit)).
- return 0.

REQ-15: mark_lock / mark_usage:
- mark_usage(curr, hlock, check):
  - For each task irq state (hardirq-on/off, softirq-on/off, read/write):
    - mark_lock(curr, hlock, LOCK_USED_IN_HARDIRQ / _IN_SOFTIRQ / _ENABLED_HARDIRQ / _ENABLED_SOFTIRQ + READ variants).
- mark_lock(curr, this, new_bit):
  - if class.usage_mask & (1 << new_bit): no-op.
  - else: class.usage_mask |= (1 << new_bit); save_trace into class.usage_traces[new_bit]; valid_state check; possibly mark_lock_irq propagation through deps.

REQ-16: lockdep_hardirqs_on_prepare / on / off / lockdep_softirqs_on / off:
- _on_prepare: validates !in_nmi, !recursion, !already-enabled, !early_boot_irqs_disabled, !lockdep_hardirq_context; records curr.hardirq_chain_key = curr.curr_chain_key; calls __trace_hardirqs_on_caller.
- on(ip): re-checks state; transitions percpu hardirqs_enabled 0→1; updates irqtrace.hardirq_enable_ip / event.
- off(ip): transitions 1→0; updates irqtrace.hardirq_disable_ip / event.
- softirqs_on(ip): if irqs_disabled WARN; transitions curr.softirqs_enabled; calls mark_held_locks(SOFTIRQ_USED_IN).
- softirqs_off(ip): transitions; updates softirq_disable_ip / event.

REQ-17: lock_release(lock, ip) / __lock_release:
- trace_lock_release.
- if !lockdep_enabled ∨ lock.key == __lockdep_no_track__: return.
- irqsave + check_flags + lockdep_recursion_inc.
- __lock_release(lock, ip):
  - depth = curr.lockdep_depth.
  - if depth ≤ 0: print_unlock_imbalance_bug; return 0.
  - hlock = find_held_lock(curr, lock, depth, &i).
  - if !hlock: print_unlock_imbalance_bug; return 0.
  - if hlock.instance == lock: lock_release_holdtime(hlock).
  - WARN(hlock.pin_count, "releasing a pinned lock").
  - if hlock.references > 0: references--; if still > 0: return 1.
  - curr.lockdep_depth = i; curr.curr_chain_key = hlock.prev_chain_key.
  - if i == depth-1: return 1.
  - reacquire_held_locks(curr, depth, i+1, &merged).

REQ-18: lock_acquired / lock_contended (lockstat only):
- __lock_acquired(lock, ip):
  - hlock = find_held_lock; if hlock.waittime_stamp: waittime = now - waittime_stamp; holdtime_stamp = now.
  - stats = get_lock_stats(class); if waittime: read_waittime / write_waittime inc; if lock.cpu ≠ cpu: bounces++; lock.cpu = cpu; lock.ip = ip.

REQ-19: lockdep_free_key_range / lockdep_reset_lock / lockdep_unregister_key:
- On module unload / kfree of code containing lock_class_key, zap classes whose `class.key` falls in [start, start+size), RCU-deferred via `free_zapped_rcu` and a `pending_free` ring (2-slot double-buffer).
- lockdep_unregister_key(key): removes key from `lock_keys_hash`; zaps classes pointing at it.

REQ-20: debug_check_no_locks_freed / no_locks_held:
- On kfree: scan all currently held locks; if a held lock's instance address falls in [from, from+len): print_freed_lock_bug; debug_locks_off.
- On task exit: if curr.lockdep_depth > 0: print_held_locks_bug.

REQ-21: lockdep_sys_exit:
- Called on syscall return; if curr.lockdep_depth > 0: report "syscall <name> returning with locks still held" + dump held locks.

REQ-22: lockdep_rcu_suspicious(file, line, s):
- Called from rcu_dereference_check / rcu_read_lock_held when suspicious; emits "WARNING: suspicious RCU usage" with held locks dump.

## Acceptance Criteria

- [ ] AC-1: `lockdep_init_map_type` rejects unregistered non-static keys with `pr_err("BUG: key %px has not been registered!")` and a `WARN_ON`.
- [ ] AC-2: `register_lock_class` interns by `(lock_class_key, subclass)`; collision returns the existing class; pool exhaustion prints `MAX_LOCKDEP_KEYS too low!` and disables lockdep.
- [ ] AC-3: `lock_acquire` is a no-op when `!debug_locks` or `lock.key == __lockdep_no_track__`; emits a `kasan_check_byte` before any other action.
- [ ] AC-4: `check_deadlock` reports recursion on same-class hold when neither read-after-read exemption nor `nest_lock` applies.
- [ ] AC-5: `check_noncircular` BFS detects cycle A→B + B→A and triggers `print_circular_bug`; lockdep self-disables (`debug_locks = 0`).
- [ ] AC-6: `check_irq_usage` rejects an edge that would link a hardirq-safe class (from prev's backward closure) to a hardirq-unsafe class (in next's forward closure).
- [ ] AC-7: `check_wait_context` rejects acquiring a class with `wait_type_inner > current_outer_wait_type` (e.g. mutex inside spinlock).
- [ ] AC-8: `validate_chain` caches by `chain_key`; subsequent identical chains skip the full check.
- [ ] AC-9: `lock_release` decrements `lockdep_depth` and restores `curr_chain_key`; unbalanced release calls `print_unlock_imbalance_bug`.
- [ ] AC-10: `lockdep_register_key` rejects static keys (`WARN_ON_ONCE(static_obj(key))`) and double-register.
- [ ] AC-11: `lockdep_hardirqs_on/off` toggle the per-CPU `hardirqs_enabled` bit, update `irqtrace.hardirq_enable_event`, and refuse to flip while `lockdep_recursion` is set.
- [ ] AC-12: `mark_lock` records the usage bit, saves a backtrace into `class.usage_traces[bit]`, and propagates through `mark_lock_irq` on first transition.
- [ ] AC-13: `debug_check_no_locks_freed(mem, len)` reports any currently-held lock whose instance address lies in `[mem, mem+len)`.
- [ ] AC-14: `lockdep_sys_exit` warns when a thread returns to userspace with `lockdep_depth > 0`.
- [ ] AC-15: `lockdep_unregister_key` queues the affected classes into `pending_free` and frees them via `free_zapped_rcu` after a grace period.

## Architecture

```
struct LockdepMap {
  key: *const LockClassKey,
  class_cache: [Option<&'static LockClass>; NR_LOCKDEP_CACHING_CLASSES],
  name: &'static str,
  wait_type_outer: u8,                                // LD_WAIT_*
  wait_type_inner: u8,
  lock_type: u8,                                      // LD_LOCK_NORMAL/_PERCPU/_WAIT_OVERRIDE
  cpu: i32,
  ip: u64,
}

struct LockClassKey {
  subkeys: [LockdepSubclassKey; MAX_LOCKDEP_SUBCLASSES],
  hash_entry: HlistNode,                              // only if dynamic
}

struct LockClass {
  key: *const LockdepSubclassKey,
  name: &'static str,
  subclass: u8,
  name_version: i32,
  hash_entry: HlistNode,
  lock_entry: ListHead,
  locks_before: ListHead,
  locks_after: ListHead,
  usage_mask: u64,
  usage_traces: [*const LockTrace; LOCK_TRACE_STATES],
  ops: u64,
  wait_type_inner: u8,
  wait_type_outer: u8,
  lock_type: u8,
  cmp_fn: Option<LockCmpFn>,
}

struct HeldLock {
  prev_chain_key: u64,
  acquire_ip: u64,
  instance: *const LockdepMap,
  nest_lock: *const LockdepMap,
  waittime_stamp: u64,
  holdtime_stamp: u64,
  class_idx: u32,
  irq_context: u8,                                    // 0=task,1=softirq,2=hardirq
  trylock: bool,
  read: u8,                                           // 0=W,1=recursive R,2=non-recursive R
  check: bool,
  hardirqs_off: bool,
  references: u32,
  pin_count: u32,
  sync: bool,
}

struct LockListEntry {
  entry: ListHead,
  class: *const LockClass,                            // target
  links_to: *const LockClass,                         // source (for class_lock_list_valid)
  trace: *const LockTrace,
  distance: u16,
  dep: u8,                                            // calc_dep result
}
```

`Lockdep::init_map_type(lock, name, key, subclass, inner, outer, lock_type)`:
1. For i in 0..NR_LOCKDEP_CACHING_CLASSES: lock.class_cache[i] = None.
2. if LOCK_STAT: lock.cpu = raw_smp_processor_id().
3. if name is empty: lock.name = "NULL"; return.
4. lock.name = name; lock.wait_type_outer = outer; lock.wait_type_inner = inner; lock.lock_type = lock_type.
5. if key is null: return.
6. if !static_obj(key) ∧ !is_dynamic_key(key): pr_err("BUG: key %px has not been registered!", key); WARN_ON(1); return.
7. lock.key = key.
8. if !debug_locks: return.
9. if subclass ≠ 0: irqsave + lockdep_recursion_inc; register_lock_class(lock, subclass, force=1); lockdep_recursion_finish; irqrestore.

`Lockdep::lock_acquire(lock, subclass, trylock, read, check, nest_lock, ip)`:
1. trace_lock_acquire(...).
2. if !debug_locks: return.
3. kasan_check_byte(lock).
4. if !lockdep_enabled():
   - if lockdep_nmi ∧ !trylock: synthesize local HeldLock { acquire_ip = ip, instance = lock, irq_context = 2, hardirqs_off = true }; verify_lock_unused.
   - return.
5. raw_local_irq_save(flags); check_flags(flags).
6. lockdep_recursion_inc.
7. lock_acquire_inner(lock, subclass, trylock, read, check, irqs_disabled_flags(flags), nest_lock, ip, references=0, pin_count=0, sync=0).
8. lockdep_recursion_finish; raw_local_irq_restore(flags).

`Lockdep::lock_acquire_inner(lock, subclass, trylock, read, check, hardirqs_off, nest_lock, ip, references, pin_count, sync) -> i32`:
1. curr = current.
2. if !debug_locks ∨ lock.key == &__lockdep_no_track__: return 0.
3. if !prove_locking ∨ lock.key == &__lockdep_no_validate__: check = 0.
4. if subclass ≥ MAX_LOCKDEP_SUBCLASSES: WARN; return 0.
5. class = if subclass < NR_LOCKDEP_CACHING_CLASSES { lock.class_cache[subclass] } else { None }.
6. if class is None: class = register_lock_class(lock, subclass, 0)?
7. depth = curr.lockdep_depth.
8. if depth ≥ MAX_LOCK_DEPTH: WARN; return 0.
9. class_idx = class.index_in_pool.
10. /* References merge */
11. if depth ∧ !sync:
    - prev = curr.held_locks[depth-1].
    - if prev.class_idx == class_idx ∧ nest_lock: prev.references += max(references, 1); return 2.
12. hlock = curr.held_locks[depth].
13. Fill hlock fields.
14. if check_wait_context(curr, hlock) == 0: return 0.
15. if !mark_usage(curr, hlock, check): return 0.
16. chain_key = curr.curr_chain_key.
17. if depth == 0: WARN_ON(chain_key ≠ INITIAL_CHAIN_KEY); chain_head = true.
18. hlock.prev_chain_key = chain_key.
19. if separate_irq_context(curr, hlock): chain_key = INITIAL_CHAIN_KEY; chain_head = true.
20. chain_key = iterate_chain_key(chain_key, hlock_id(hlock)).
21. if nest_lock ∧ !__lock_is_held(nest_lock, -1): print_lock_nested_lock_not_held; return 0.
22. if !validate_chain(curr, hlock, chain_head, chain_key): return 0.
23. if sync: return 1.
24. curr.curr_chain_key = chain_key; curr.lockdep_depth++.
25. check_chain_key(curr).
26. if curr.lockdep_depth ≥ MAX_LOCK_DEPTH: debug_locks_off; print "BUG: MAX_LOCK_DEPTH too low"; return 0.
27. return 1.

`Lockdep::check_deadlock(curr, next) -> i32`:
1. nest = None.
2. For i in 0..curr.lockdep_depth:
   - prev = curr.held_locks[i].
   - if prev.instance == next.nest_lock: nest = Some(prev).
   - if hlock_class(prev) ≠ hlock_class(next): continue.
   - if next.read == 2 ∧ prev.read ≠ 0: continue.   // recursive read exemption
   - class = hlock_class(prev).
   - if class.cmp_fn(prev.instance, next.instance) < 0: continue.
   - if nest.is_some(): return 2.
   - print_deadlock_bug(curr, prev, next); return 0.
3. return 1.

`Lockdep::check_noncircular(src, target, trace_out) -> BfsResult`:
1. BFS forwards from src via `bfs_init_root + __bfs_forwards` over `LockClass::locks_after` lists.
2. Predicate `hlock_conflict(entry, target)` ⟹ BFS_RMATCH (cycle detected).
3. Capacity bound: MAX_CIRCULAR_QUEUE_SIZE; overflow ⟹ BFS_EQUEUEFULL.
4. If BFS_RMATCH or bfs_error ⟹ print_circular_bug(this, target, src.parent_walk, *trace_out).

`Lockdep::check_irq_usage(curr, prev, next)`:
1. usage_mask = 0; bfs_init_rootb(&this, prev).
2. __bfs_backwards(&this, &usage_mask, usage_accumulate, usage_skip, NULL).
3. usage_mask &= LOCKF_USED_IN_IRQ_ALL.
4. if usage_mask == 0: return 1.
5. forward_mask = exclusive_mask(usage_mask).
6. bfs_init_root(&that, next); find_usage_forwards(&that, forward_mask, &fwd_entry).
7. if BFS_RNOMATCH: return 1.
8. backward_mask = original_mask(fwd_entry.class.usage_mask & LOCKF_ENABLED_IRQ_ALL).
9. find_usage_backwards(&this, backward_mask, &bwd_entry).
10. find_exclusive_match(bwd_entry.class.usage_mask, fwd_entry.class.usage_mask, &backward_bit, &forward_bit).
11. print_bad_irq_dependency(...); return 0.

`Lockdep::lock_release_inner(lock, ip)`:
1. depth = curr.lockdep_depth.
2. if depth ≤ 0: print_unlock_imbalance_bug; return 0.
3. hlock = find_held_lock(curr, lock, depth, &i)?
4. if hlock.instance == lock: lock_release_holdtime(hlock).
5. WARN(hlock.pin_count > 0, "releasing a pinned lock").
6. if hlock.references > 0: hlock.references -= 1; if still > 0: return 1.
7. curr.lockdep_depth = i; curr.curr_chain_key = hlock.prev_chain_key.
8. if i == depth - 1: return 1.
9. reacquire_held_locks(curr, depth, i+1, &merged) — replay locks above i.

`Lockdep::register_key(key)`:
1. WARN_ON_ONCE(static_obj(key)).
2. hash_head = keyhashentry(key).
3. raw_local_irq_save; graph_lock?
4. For k in hash_head: WARN_ON_ONCE(k == key).
5. hlist_add_head_rcu(key.hash_entry, hash_head); nr_dynamic_keys += 1.
6. graph_unlock; raw_local_irq_restore.

`Lockdep::hardirqs_on(ip)` (noinstr):
1. if !debug_locks: return.
2. if in_nmi(): if !TRACE_IRQFLAGS_NMI: return; goto skip_checks.
3. if this_cpu_read(lockdep_recursion): return.
4. if lockdep_hardirqs_enabled(): __debug_atomic_inc(redundant_hardirqs_on); return.
5. WARN_ON(!irqs_disabled()).
6. DEBUG_LOCKS_WARN_ON(curr.hardirq_chain_key ≠ curr.curr_chain_key).
7. skip_checks: __this_cpu_write(hardirqs_enabled, 1); trace.hardirq_enable_ip = ip; trace.hardirq_enable_event += 1; debug_atomic_inc(hardirqs_on_events).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `class_pool_bounded` | INVARIANT | `register_lock_class`: nr_lock_classes ≤ MAX_LOCKDEP_KEYS. |
| `held_locks_depth_bounded` | INVARIANT | per-`__lock_acquire`: curr.lockdep_depth < MAX_LOCK_DEPTH. |
| `subclass_bounded` | INVARIANT | subclass < MAX_LOCKDEP_SUBCLASSES. |
| `chain_key_zero_when_empty` | INVARIANT | curr.lockdep_depth == 0 ⟹ curr.curr_chain_key == INITIAL_CHAIN_KEY. |
| `release_pops_top` | INVARIANT | post-`__lock_release` on innermost hlock: lockdep_depth -= 1. |
| `register_key_no_static` | INVARIANT | `lockdep_register_key`: static_obj(key) ⟹ early return (WARN_ON_ONCE). |
| `init_map_rejects_unregistered_nonstatic` | INVARIANT | `lockdep_init_map_type`: !static_obj(key) ∧ !is_dynamic_key(key) ⟹ no graph mutation. |
| `irq_state_recursion_guard` | INVARIANT | `lockdep_hardirqs_on`: lockdep_recursion ⟹ no-op (no per-CPU write). |
| `validate_chain_idempotent` | INVARIANT | per-`validate_chain`: same chain_key ⟹ deterministic result. |
| `no_locks_held_at_task_exit` | INVARIANT | `debug_check_no_locks_held`: lockdep_depth must be 0 on exit. |

### Layer 2: TLA+

`kernel/locking/lockdep.tla`:
- State: { curr.lockdep_depth, curr.curr_chain_key, lock_classes[], all_lock_classes, classhash_table, chainhash_table, per_cpu_hardirqs_enabled, current_softirqs_enabled }.
- Per-acquire / per-release transitions atomic under `graph_lock` (when graph mutation needed) and `raw_local_irq_save`.
- Properties:
  - `safety_no_cycles_added` — `check_prev_add` rejects edge prev→next whenever the existing graph has next→...→prev (BFS witness).
  - `safety_irq_safe_unsafe_never_chained` — no edge connects a class with LOCK_USED_IN_HARDIRQ to a class with LOCK_ENABLED_HARDIRQ along a single chain.
  - `safety_wait_type_monotone` — per-chain wait_type_inner is non-increasing from outer-most to inner-most held lock.
  - `safety_recursion_guard` — `lockdep_recursion` set ⟹ subsequent reentrant lockdep entry points return immediately.
  - `safety_dep_dup_no_dup_edges` — no two list_entries with identical (links_to, class).
  - `safety_unbalanced_release_reports` — release without matching held entry sets `debug_locks = 0` after `print_unlock_imbalance_bug`.
  - `safety_lockdep_off_is_sticky` — once `debug_locks = 0`, no further graph mutation.
  - `liveness_per_chain_caches` — first acquire of a chain validates; subsequent same chains short-circuit via `chainhash_table` hit.
  - `liveness_dynamic_key_zap` — `lockdep_unregister_key` eventually frees all classes referencing the key (after RCU grace period).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `register_lock_class` post: returns class with class.key == lock.key.subkeys+subclass ∨ NULL | `Lockdep::register_lock_class` |
| `check_deadlock` post: returns 1 ⟹ no same-class hold without exemption; returns 0 ⟹ print_deadlock_bug invoked | `Lockdep::check_deadlock` |
| `check_noncircular` post: BFS_RMATCH ⟹ existing path from src to target in `locks_after` graph | `Lockdep::check_noncircular` |
| `check_irq_usage` post: ret==0 ⟹ ∃ backward bit b in {USED_IN_*} ∧ forward bit f in {ENABLED_*} with exclusive(b)==f | `Lockdep::check_irq_usage` |
| `mark_lock` post: class.usage_mask & (1<<new_bit) == 1 ∧ class.usage_traces[new_bit] ≠ NULL | `Lockdep::mark_lock` |
| `lock_acquire` post: curr.lockdep_depth' == curr.lockdep_depth + 1 ∨ early-return | `Lockdep::lock_acquire` |
| `lock_release` post: curr.lockdep_depth' < curr.lockdep_depth ∨ references-decrement | `Lockdep::lock_release` |
| `lockdep_register_key` post: key ∈ lock_keys_hash ∧ nr_dynamic_keys += 1 | `Lockdep::register_key` |
| `lockdep_hardirqs_on` post: per_cpu.hardirqs_enabled == 1 ∧ trace.hardirq_enable_ip == ip | `Lockdep::hardirqs_on` |
| `lockdep_unregister_key` post: key removed from lock_keys_hash; pending classes queued for free_zapped_rcu | `Lockdep::unregister_key` |

### Layer 4: Verus/Creusot functional

`Per-acquire flow: trace_lock_acquire → kasan_check_byte → lockdep_recursion_inc → __lock_acquire (register_lock_class → push HeldLock → mark_usage → validate_chain (check_deadlock, check_prevs_add → check_prev_add → check_noncircular + check_irq_usage)) → lockdep_recursion_finish` semantic equivalence to upstream per-`Documentation/locking/lockdep-design.rst` (class/key/chain model + irq-safety state machine + wait-type hierarchy).

## Hardening

(Inherits row-1 features from `kernel/locking/00-overview.md` § Hardening.)

Lockdep reinforcement:

- **Per-class pool bounded by MAX_LOCKDEP_KEYS** — defense against per-runaway class-allocation; pool exhaustion disables lockdep with explicit `MAX_LOCKDEP_KEYS too low!`.
- **Per-edge pool bounded by MAX_LOCKDEP_ENTRIES** — defense against per-graph blow-up.
- **Per-chain pool bounded by MAX_LOCKDEP_CHAINS** — defense against per-chain-cache blow-up.
- **Per-held-lock depth bounded by MAX_LOCK_DEPTH** — defense against per-task lock-stack overflow.
- **Per-BFS bounded by MAX_CIRCULAR_QUEUE_SIZE** — defense against per-search OOM on adversarial graphs.
- **Per-static_obj() + is_dynamic_key() gate** — defense against using an unregistered key (use-after-free of class identity).
- **Per-lockdep_recursion percpu guard** — defense against re-entrant lockdep calls from within lockdep itself.
- **Per-graph_lock under raw_local_irq_save** — defense against per-cross-CPU graph corruption.
- **Per-debug_locks sticky off** — defense against further graph mutation after the first reported bug.
- **Per-kasan_check_byte at lock_acquire entry** — defense against use-after-free of the LockdepMap itself.
- **Per-debug_check_no_locks_freed** — defense against per-kfree-while-held catastrophe.
- **Per-debug_check_no_locks_held at task exit** — defense against per-leaked-lock task exit.
- **Per-lockdep_sys_exit at syscall return** — defense against per-syscall-held-lock kernel→user leak.
- **Per-print_lockdep_off + dump_stack on first bug** — defense against silent corruption (preserves forensic record).
- **Per-pending_free RCU-deferred zap** — defense against per-class-free use-after-free.

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
- **lock_class_key under PAX_RAP** — static keys treated as code-pointer-adjacent metadata; tampered keys fail signature check before being installed into the class hash.
- **lockdep_init_map call sites** — indirect calls into per-class init helpers carry kCFI signatures matching `lockdep_init_map_type`.
- **Class hash table bounds** — `classhash_table` index masking validated against `CLASSHASH_SIZE` to prevent OOB read during PAX_UDEREF transitions.
- **Chain hash collision dump** — chain hash dump in /proc gated by GRKERNSEC_HIDESYM so `class->key` pointers are sanitized.
- **lockdep_off/on counters** — per-task `lockdep_recursion` under PAX_REFCOUNT saturating semantics; runaway recursion saturates rather than wraps.
- **Pending-free RCU callback** — `free_zapped_classes` invocation carries PAX_RAP signature; mismatched callback aborts before touching the class array.

Per-doc rationale: lockdep's class keys, chain hashes, and per-task recursion counters are high-value targets for an attacker seeking either silent disablement of all kernel deadlock checking or a controllable indirect-call gadget through the class-zap RCU path. PaX_RAP/kCFI on key-init and callback paths, plus PAX_REFCOUNT on the recursion counters, close those gadgets while GRKERNSEC_HIDESYM keeps the class pointers themselves out of debugfs and /proc leaks.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- spinlock_t / raw_spinlock_t implementation — covered in `kernel/locking/spinlock.md` Tier-3
- mutex implementation — covered in `kernel/locking/mutex.md` Tier-3
- rwsem implementation — covered in `kernel/locking/rwsem.md` Tier-3
- qspinlock — covered in `kernel/locking/qspinlock.md` Tier-3
- lockdep_proc / /proc/lockdep formatting (covered separately if expanded)
- lockdep_selftest.c boot-time selftest harness (covered separately if expanded)
- Lock-statistics rendering (`/proc/lock_stat`) — separate doc if expanded
- Klogd / nbcon emergency console paths — covered in `kernel/printk/` Tier-3 (if expanded)
- Implementation code
