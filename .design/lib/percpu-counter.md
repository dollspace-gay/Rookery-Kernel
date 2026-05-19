# Tier-3: lib/percpu_counter.c — Batched per-CPU counter primitive

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: lib/00-overview.md
upstream-paths:
  - lib/percpu_counter.c (~407 lines)
  - include/linux/percpu_counter.h
  - include/linux/debugobjects.h (CONFIG_DEBUG_OBJECTS_PERCPU_COUNTER hooks)
-->

## Summary

`percpu_counter` is the kernel's general-purpose **batched approximate-then-precise counter**. It is the answer to the question "I have a counter that is updated extremely often from many CPUs, but readers tolerate a small bounded error — how do I avoid contending an atomic across all sockets?" Per-counter, a small `s32` per-CPU shard holds *local* deltas. When `|local + amount|` would exceed `batch` (default 32, but scaled at hotplug to `max(32, 2 * num_online_cpus)`), the local shard is folded into the global `fbc->count` (an `s64` protected by `fbc->lock`) under that lock; otherwise the update is a fast `this_cpu_*` op. Per-`percpu_counter_read`: returns the **approximate** global value (just `fbc->count`, ignoring shards) — fast but with up to `batch * num_online_cpus` error. Per-`percpu_counter_sum`: walks all online + dying CPUs under `fbc->lock` and returns the **precise** value — slow but exact at that instant. Per-`percpu_counter_set`: forcibly zeros every shard and assigns `fbc->count = amount` under the lock. Per-`percpu_counter_sync`: folds the local shard into the global under the lock without changing the total (used to reduce drift after a batch-tuning change). Per-CPU-hotplug: when a CPU goes dead, its shard is folded into every registered counter's global so the bookkeeping is preserved. Per-`__percpu_counter_compare` and `__percpu_counter_limited_add`: amortize cost by first using the approximate value, then upgrading to a precise sum *only* when the approximate value is too close to the threshold to make a confident decision. Critical for: mm/slab object counts, block-layer per-bdi statistics (`WB_RECLAIMABLE`, `WB_WRITEBACK`, `WB_DIRTIED`, `WB_WRITTEN`), `vm_committed_as`, NFS file-descriptor counters, fs-specific inode/dquot counters.

This Tier-3 covers `lib/percpu_counter.c` (~407 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct percpu_counter` | per-counter struct | `PercpuCounter` |
| `percpu_counter_init()` (`__percpu_counter_init_many` with n=1) | per-counter init | `PercpuCounter::init` |
| `__percpu_counter_init_many()` | per-counter-array init | `PercpuCounter::init_many` |
| `percpu_counter_destroy()` (`percpu_counter_destroy_many` with n=1) | per-counter teardown | `PercpuCounter::destroy` |
| `percpu_counter_destroy_many()` | per-counter-array teardown | `PercpuCounter::destroy_many` |
| `percpu_counter_set()` | per-counter zero-and-set | `PercpuCounter::set` |
| `percpu_counter_add()` (`__percpu_counter_add` with default batch) | per-counter add | `PercpuCounter::add` |
| `percpu_counter_sub()` (calls add with -amount) | per-counter sub | `PercpuCounter::sub` |
| `percpu_counter_add_batch()` | per-counter add with custom batch | `PercpuCounter::add_batch` |
| `percpu_counter_sync()` | per-counter local-fold | `PercpuCounter::sync` |
| `percpu_counter_read()` | per-counter approximate read | `PercpuCounter::read` |
| `percpu_counter_read_positive()` | per-counter approximate, clamped ≥ 0 | `PercpuCounter::read_positive` |
| `__percpu_counter_sum()` | per-counter precise sum | `PercpuCounter::sum` |
| `percpu_counter_sum_positive()` | per-counter precise, clamped ≥ 0 | `PercpuCounter::sum_positive` |
| `__percpu_counter_compare()` | per-counter approx-then-precise compare | `PercpuCounter::compare` |
| `__percpu_counter_limited_add()` | per-counter bounded add | `PercpuCounter::limited_add` |
| `percpu_counter_batch` (extern int) | per-counter global batch tunable | `PERCPU_COUNTER_BATCH` static |
| `compute_batch_value()` | per-CPU-online callback recomputing batch | `PercpuCounter::compute_batch` |
| `percpu_counter_cpu_dead()` | per-CPU-dead fold-shard callback | `PercpuCounter::cpu_dead` |
| `percpu_counter_startup()` (`__init`) | per-subsystem cpuhp wiring | `PercpuCounter::startup` |
| `percpu_counter_fixup_free()` | per-debug_obj UAF fixup | `PercpuCounter::debug_fixup_free` |
| `debug_percpu_counter_activate()` / `_deactivate()` | per-debugobj lifecycle | `PercpuCounter::debug_activate` / `debug_deactivate` |
| `percpu_counters` (list_head) | per-subsystem registered-counters list | `PERCPU_COUNTERS_LIST` static |
| `percpu_counters_lock` (spinlock) | per-subsystem registered-counters serializer | `PERCPU_COUNTERS_LOCK` static |

## Compatibility contract

REQ-1: struct percpu_counter (per `include/linux/percpu_counter.h`):
- lock: raw_spinlock_t — protects `count` and the per-CPU fold operations.
- count: s64 — the *global* portion of the value; precise value is `count + Σ counters[cpu]`.
- list: list_head — linkage on `percpu_counters` when `CONFIG_HOTPLUG_CPU` (so CPU-dead callback can fold shards).
- counters: `s32 __percpu *` — per-CPU shard pointer (one s32 per possible CPU, allocated via `__alloc_percpu_gfp`).

REQ-2: __percpu_counter_init_many(fbc, amount, gfp, nr_counters, key):
- counter_size = ALIGN(sizeof(s32), __alignof__(s32)).
- counters = __alloc_percpu_gfp(nr_counters * counter_size, __alignof__(s32), gfp).
- if !counters: fbc[0].counters = NULL; return -ENOMEM.
- for i in [0, nr_counters):
  - raw_spin_lock_init(&fbc[i].lock).
  - lockdep_set_class(&fbc[i].lock, key).
  - if CONFIG_HOTPLUG_CPU: INIT_LIST_HEAD(&fbc[i].list).
  - fbc[i].count = amount.
  - fbc[i].counters = (s32 __percpu *)(counters_base) + i * counter_size.
  - debug_percpu_counter_activate(&fbc[i]).
- if CONFIG_HOTPLUG_CPU:
  - spin_lock_irqsave(&percpu_counters_lock).
  - for each i: list_add(&fbc[i].list, &percpu_counters).
  - spin_unlock_irqrestore.
- return 0.

REQ-3: percpu_counter_destroy_many(fbc, nr_counters):
- WARN_ON_ONCE(!fbc); return.
- if !fbc[0].counters: return (uninitialised — safe no-op).
- for i in [0, nr_counters): debug_percpu_counter_deactivate(&fbc[i]).
- if CONFIG_HOTPLUG_CPU:
  - spin_lock_irqsave(&percpu_counters_lock).
  - for each i: list_del(&fbc[i].list).
  - spin_unlock_irqrestore.
- free_percpu(fbc[0].counters).
- for each i: fbc[i].counters = NULL.

REQ-4: percpu_counter_set(fbc, amount):
- raw_spin_lock_irqsave(&fbc->lock).
- for each possible_cpu: *per_cpu_ptr(fbc->counters, cpu) = 0.
- fbc->count = amount.
- raw_spin_unlock_irqrestore.

REQ-5: percpu_counter_add_batch(fbc, amount, batch) — fast-path branch (CONFIG_HAVE_CMPXCHG_LOCAL):
- count = this_cpu_read(*fbc->counters).
- loop:
  - if abs(count + amount) >= batch:
    - raw_spin_lock_irqsave(&fbc->lock).
    - count = __this_cpu_read(*fbc->counters) /* may have migrated */.
    - fbc->count += count + amount.
    - __this_cpu_sub(*fbc->counters, count) /* zero shard */.
    - raw_spin_unlock_irqrestore.
    - return.
  - if this_cpu_try_cmpxchg(*fbc->counters, &count, count + amount): return.

REQ-6: percpu_counter_add_batch(fbc, amount, batch) — fallback branch (!CONFIG_HAVE_CMPXCHG_LOCAL):
- local_irq_save(flags).
- count = __this_cpu_read(*fbc->counters) + amount.
- if abs(count) >= batch:
  - raw_spin_lock(&fbc->lock).
  - fbc->count += count.
  - __this_cpu_sub(*fbc->counters, count - amount).
  - raw_spin_unlock(&fbc->lock).
- else: this_cpu_add(*fbc->counters, amount).
- local_irq_restore(flags).

REQ-7: percpu_counter_sync(fbc):
- raw_spin_lock_irqsave(&fbc->lock).
- count = __this_cpu_read(*fbc->counters).
- fbc->count += count.
- __this_cpu_sub(*fbc->counters, count) /* zero local shard */.
- raw_spin_unlock_irqrestore.

REQ-8: __percpu_counter_sum(fbc) (precise):
- raw_spin_lock_irqsave(&fbc->lock).
- ret = fbc->count.
- for cpu in (cpu_online_mask | cpu_dying_mask): ret += *per_cpu_ptr(fbc->counters, cpu).
- raw_spin_unlock_irqrestore.
- return ret.

REQ-9: percpu_counter_read (inline in header):
- return fbc->count (approximate; ignores shards).

REQ-10: percpu_counter_read_positive (inline in header):
- ret = percpu_counter_read(fbc).
- if ret > 0: return ret else return 0.

REQ-11: __percpu_counter_compare(fbc, rhs, batch):
- count = percpu_counter_read(fbc) /* approximate */.
- if abs(count - rhs) > batch * num_online_cpus():
  - /* approximate is clearly distinguishable */
  - return (count > rhs ? 1 : -1).
- /* fall back to precise sum */
- count = percpu_counter_sum(fbc).
- return (count > rhs ? 1 : (count < rhs ? -1 : 0)).

REQ-12: __percpu_counter_limited_add(fbc, limit, amount, batch):
- if amount == 0: return true.
- local_irq_save(flags).
- unknown = batch * num_online_cpus().
- count = __this_cpu_read(*fbc->counters).
- /* Fast path: no lock if approx is clearly within / outside limit */
- if abs(count + amount) <= batch ∧ ((amount > 0 ∧ fbc->count + unknown ≤ limit) ∨ (amount < 0 ∧ fbc->count - unknown ≥ limit)):
  - this_cpu_add(*fbc->counters, amount); local_irq_restore; return true.
- raw_spin_lock(&fbc->lock).
- count = fbc->count + amount.
- /* Conservative check using ±unknown bound */
- if amount > 0:
  - if count - unknown > limit: goto out (good=false).
  - if count + unknown ≤ limit: good = true.
- else:
  - if count + unknown < limit: goto out (good=false).
  - if count - unknown ≥ limit: good = true.
- if !good:
  - /* Precise: fold all shards into count */
  - for cpu in (online | dying): count += *per_cpu_ptr(fbc->counters, cpu).
  - if (amount > 0 ∧ count > limit) ∨ (amount < 0 ∧ count < limit): goto out (good=false).
  - good = true.
- /* Commit local shard into global */
- count = __this_cpu_read(*fbc->counters).
- fbc->count += count + amount.
- __this_cpu_sub(*fbc->counters, count).
- out:
- raw_spin_unlock(&fbc->lock); local_irq_restore.
- return good.

REQ-13: compute_batch_value(cpu) — cpuhp ONLINE callback:
- nr = num_online_cpus().
- percpu_counter_batch = max(32, nr * 2).
- return 0.

REQ-14: percpu_counter_cpu_dead(cpu) — cpuhp DEAD callback:
- compute_batch_value(cpu) /* refresh */.
- if CONFIG_HOTPLUG_CPU:
  - spin_lock_irq(&percpu_counters_lock).
  - for each fbc in &percpu_counters:
    - raw_spin_lock(&fbc->lock).
    - pcount = per_cpu_ptr(fbc->counters, cpu).
    - fbc->count += *pcount.
    - *pcount = 0.
    - raw_spin_unlock.
  - spin_unlock_irq.

REQ-15: percpu_counter_startup() (__init):
- cpuhp_setup_state(CPUHP_AP_ONLINE_DYN, "lib/percpu_cnt:online", compute_batch_value, NULL).
- cpuhp_setup_state_nocalls(CPUHP_PERCPU_CNT_DEAD, "lib/percpu_cnt:dead", NULL, percpu_counter_cpu_dead).

REQ-16: CONFIG_DEBUG_OBJECTS_PERCPU_COUNTER:
- percpu_counter_fixup_free(addr, ODEBUG_STATE_ACTIVE): percpu_counter_destroy(fbc); debug_object_free; return true.
- debug_percpu_counter_activate: debug_object_init + debug_object_activate.
- debug_percpu_counter_deactivate: debug_object_deactivate + debug_object_free.
- Detects "free of active percpu_counter" UAF / leak.

REQ-17: Interrupt safety (per per-architecture branch):
- HAVE_CMPXCHG_LOCAL: fast path uses *non-locked* local cmpxchg; safe vs IRQ on same CPU because cmpxchg-loop retries.
- !HAVE_CMPXCHG_LOCAL: fast path uses local_irq_save/restore.
- Both: slow path holds raw_spinlock with IRQs disabled.

REQ-18: Read approximation bound:
- |percpu_counter_read(fbc) - precise| ≤ batch * num_online_cpus.
- For default batch 32 and 256 CPUs: ≤ 8192 (~32 KiB).

## Acceptance Criteria

- [ ] AC-1: percpu_counter_init zero amount: read returns 0; sum returns 0.
- [ ] AC-2: percpu_counter_add by < batch on one CPU: read still returns initial; sum returns initial + amount.
- [ ] AC-3: percpu_counter_add accumulating to |local| ≥ batch: read advances by ≈ local fold; sum exact.
- [ ] AC-4: percpu_counter_set(N): read returns N; sum returns N; all shards zeroed.
- [ ] AC-5: percpu_counter_sync: shard zeroed, total preserved.
- [ ] AC-6: percpu_counter_destroy: counters = NULL; subsequent destroy no-op.
- [ ] AC-7: percpu_counter_destroy on never-inited fbc (fbc[0].counters == NULL): no-op + WARN-free.
- [ ] AC-8: __percpu_counter_compare with rhs far from count (abs diff > batch * cpus): no precise sum (read-only).
- [ ] AC-9: __percpu_counter_compare with rhs close to count: precise sum invoked.
- [ ] AC-10: __percpu_counter_limited_add over-limit: returns false, no mutation.
- [ ] AC-11: __percpu_counter_limited_add within-limit fast: returns true, only shard mutated.
- [ ] AC-12: __percpu_counter_limited_add boundary: precise-sum fold then decision.
- [ ] AC-13: CPU offline: shard folded into fbc->count, next read returns prior precise value.
- [ ] AC-14: compute_batch_value at hotplug: percpu_counter_batch == max(32, 2 * num_online_cpus).
- [ ] AC-15: Concurrent add on N CPUs to same counter: sum is exact (no lost updates).

## Architecture

```
struct PercpuCounter {
  lock: RawSpinLock,
  count: i64,                        // global portion
  list: ListHead,                    // linkage on PERCPU_COUNTERS_LIST (cfg(HOTPLUG_CPU))
  counters: PercpuPtr<i32>,          // per-CPU shards
}

static PERCPU_COUNTERS_LIST: ListHead;             // cfg(HOTPLUG_CPU)
static PERCPU_COUNTERS_LOCK: SpinLock;             // cfg(HOTPLUG_CPU); protects the list
static PERCPU_COUNTER_BATCH: AtomicI32 = 32;       // tuned by cpuhp ONLINE
```

`PercpuCounter::init_many(fbc: &mut [PercpuCounter], amount: i64, gfp: Gfp, key: &LockClassKey) -> Result<()>`:
1. let counter_size = align_of::<i32>().max(size_of::<i32>()).
2. let counters_base = alloc_percpu_gfp(fbc.len() * counter_size, align_of::<i32>(), gfp).ok_or(ENOMEM)?.
3. for (i, f) in fbc.iter_mut().enumerate():
   - raw_spin_lock_init(&f.lock).
   - lockdep_set_class(&f.lock, key).
   - if cfg!(HOTPLUG_CPU): INIT_LIST_HEAD(&f.list).
   - f.count = amount.
   - f.counters = counters_base.byte_offset(i * counter_size).cast().
   - debug_percpu_counter_activate(f).
4. if cfg!(HOTPLUG_CPU):
   - PERCPU_COUNTERS_LOCK.lock_irqsave().
   - for f in fbc: list_add(&f.list, &PERCPU_COUNTERS_LIST).
   - unlock.
5. Ok(())

`PercpuCounter::destroy_many(fbc: &mut [PercpuCounter])`:
1. if fbc.is_empty(): return.  // logically WARN-not-empty by caller, mirrors WARN_ON_ONCE
2. if fbc[0].counters.is_null(): return.
3. for f in fbc: debug_percpu_counter_deactivate(f).
4. if cfg!(HOTPLUG_CPU):
   - PERCPU_COUNTERS_LOCK.lock_irqsave().
   - for f in fbc: list_del(&f.list).
   - unlock.
5. free_percpu(fbc[0].counters).
6. for f in fbc: f.counters = ptr::null_mut().

`PercpuCounter::set(fbc, amount: i64)`:
1. let _g = fbc.lock.lock_irqsave().
2. for cpu in possible_cpus(): *per_cpu_ptr(fbc.counters, cpu) = 0.
3. fbc.count = amount.

`PercpuCounter::add_batch(fbc, amount: i64, batch: i32)` (CONFIG_HAVE_CMPXCHG_LOCAL):
1. let mut count: i64 = this_cpu_read(fbc.counters).
2. loop:
   a. if (count + amount).abs() ≥ batch as i64:
      - let _g = fbc.lock.lock_irqsave().
      - let cur = __this_cpu_read(fbc.counters).
      - fbc.count += cur + amount.
      - __this_cpu_sub(fbc.counters, cur).
      - return.
   b. if this_cpu_try_cmpxchg(fbc.counters, &mut count, count + amount): return.

`PercpuCounter::add_batch(fbc, amount, batch)` (!CONFIG_HAVE_CMPXCHG_LOCAL):
1. local_irq_save(flags).
2. let count = __this_cpu_read(fbc.counters) + amount.
3. if count.abs() ≥ batch as i64:
   - fbc.lock.lock().
   - fbc.count += count.
   - __this_cpu_sub(fbc.counters, count - amount).
   - fbc.lock.unlock().
4. else: this_cpu_add(fbc.counters, amount).
5. local_irq_restore(flags).

`PercpuCounter::sync(fbc)`:
1. let _g = fbc.lock.lock_irqsave().
2. let count = __this_cpu_read(fbc.counters).
3. fbc.count += count.
4. __this_cpu_sub(fbc.counters, count).

`PercpuCounter::sum(fbc) -> i64`:
1. let _g = fbc.lock.lock_irqsave().
2. let mut ret = fbc.count.
3. for cpu in cpu_online_mask().union(cpu_dying_mask()):
   - ret += *per_cpu_ptr(fbc.counters, cpu) as i64.
4. ret

`PercpuCounter::compare(fbc, rhs: i64, batch: i32) -> Ordering`:
1. let approx = PercpuCounter::read(fbc).
2. if (approx - rhs).abs() > (batch as i64) * (num_online_cpus() as i64):
   - return if approx > rhs { Greater } else { Less }.
3. let precise = PercpuCounter::sum(fbc).
4. precise.cmp(&rhs)

`PercpuCounter::limited_add(fbc, limit: i64, amount: i64, batch: i32) -> bool`:
1. if amount == 0: return true.
2. local_irq_save(flags).
3. let unknown = (batch as i64) * (num_online_cpus() as i64).
4. let mut count = __this_cpu_read(fbc.counters) as i64.
5. /* Fast path: shard delta still within batch AND clearly within / outside limit */
6. if (count + amount).abs() ≤ (batch as i64)
   ∧ ((amount > 0 ∧ fbc.count + unknown ≤ limit) ∨ (amount < 0 ∧ fbc.count - unknown ≥ limit)):
   - this_cpu_add(fbc.counters, amount).
   - local_irq_restore.
   - return true.
7. fbc.lock.lock().
8. count = fbc.count + amount.
9. let mut good = false.
10. if amount > 0:
    - if count - unknown > limit: goto out.
    - if count + unknown ≤ limit: good = true.
11. else:
    - if count + unknown < limit: goto out.
    - if count - unknown ≥ limit: good = true.
12. if !good:
    - /* Precise: fold all shards into the candidate */
    - for cpu in cpu_online_mask().union(cpu_dying_mask()):
      - count += *per_cpu_ptr(fbc.counters, cpu) as i64.
    - if (amount > 0 ∧ count > limit) ∨ (amount < 0 ∧ count < limit): goto out.
    - good = true.
13. /* Commit local shard into global */
14. let local = __this_cpu_read(fbc.counters) as i64.
15. fbc.count += local + amount.
16. __this_cpu_sub(fbc.counters, local).
17. out:
18. fbc.lock.unlock(); local_irq_restore(flags).
19. good

`PercpuCounter::cpu_dead(cpu)`:
1. PercpuCounter::compute_batch(cpu).
2. if cfg!(HOTPLUG_CPU):
   - PERCPU_COUNTERS_LOCK.lock_irq().
   - for fbc in PERCPU_COUNTERS_LIST.iter():
     - let _g = fbc.lock.lock().
     - let pcount = per_cpu_ptr(fbc.counters, cpu).
     - fbc.count += *pcount.
     - *pcount = 0.
   - PERCPU_COUNTERS_LOCK.unlock_irq().

`PercpuCounter::compute_batch(cpu) -> i32`:
1. let nr = num_online_cpus().
2. PERCPU_COUNTER_BATCH.store(max(32, nr as i32 * 2)).
3. 0

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `add_batch_no_irq_reentrancy` | INVARIANT | per-add_batch: !HAVE_CMPXCHG_LOCAL branch enters with IRQs disabled; HAVE_CMPXCHG_LOCAL branch's slow path also IRQ-disabled. |
| `slow_path_holds_fbc_lock` | INVARIANT | per-add_batch slow path: fbc->lock held during fbc->count += and __this_cpu_sub. |
| `sum_walks_online_and_dying` | INVARIANT | per-sum: iteration covers cpu_online_mask | cpu_dying_mask. |
| `set_zeros_all_possible` | INVARIANT | per-set: every possible CPU's shard cleared. |
| `init_zero_on_oom` | INVARIANT | per-init_many: ENOMEM ⟹ fbc[0].counters == NULL (so destroy is safe). |
| `destroy_idempotent` | INVARIANT | per-destroy_many: fbc[0].counters == NULL on entry ⟹ no-op. |
| `cmpxchg_loop_terminates` | INVARIANT | per-add_batch HAVE_CMPXCHG branch: loop progresses on cmpxchg fail (count rewritten by caller side). |
| `list_link_under_global_lock` | INVARIANT | per-init/destroy_many: percpu_counters list_add/del under percpu_counters_lock. |
| `cpu_dead_locks_outer_then_inner` | INVARIANT | per-cpu_dead: percpu_counters_lock outer, fbc->lock inner — no deadlock. |
| `limited_add_irq_balanced` | INVARIANT | per-limited_add: every return path matches local_irq_save with local_irq_restore. |
| `read_does_not_lock` | INVARIANT | per-percpu_counter_read: returns fbc->count without locking. |

### Layer 2: TLA+

`lib/percpu_counter.tla`:
- Per-CPU N producers calling add_batch concurrently.
- Per-precise reader (sum / compare-with-fall-through).
- Per-CPU hotplug DEAD callback folding shard.
- Properties:
  - `safety_precise_sum_invariant` — at any quiescent point: sum(fbc) == fbc.count + Σ shards.
  - `safety_approx_bound` — |read(fbc) - sum(fbc)| ≤ batch * |online ∪ dying|.
  - `safety_no_lost_update` — Σ amounts contributed by all add_batch calls == sum(fbc) - initial.
  - `safety_set_resets_all_shards` — after set(N): sum(fbc) == N and every shard == 0.
  - `safety_cpu_dead_preserves_total` — pre vs post CPU-dead: sum unchanged.
  - `safety_limited_add_respects_limit` — limited_add returns true ⟹ post-condition sum within limit.
  - `safety_limited_add_no_mutation_on_false` — limited_add returns false ⟹ sum unchanged.
  - `safety_destroy_after_no_add` — destroy ⟹ no subsequent add (caller contract; modeled as assumption).
  - `liveness_add_batch_terminates` — every add_batch returns.
  - `liveness_sum_terminates` — sum returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `init_many` post: every fbc[i].lock initialised; fbc[i].counters == base + i*counter_size or NULL | `PercpuCounter::init_many` |
| `destroy_many` post: fbc[i].counters == NULL ∀ i; free_percpu called once | `PercpuCounter::destroy_many` |
| `set` post: ∀ cpu, *per_cpu_ptr(fbc.counters, cpu) == 0 ∧ fbc.count == amount | `PercpuCounter::set` |
| `add_batch` post: sum(fbc) increased by amount; fbc->lock held only during fold | `PercpuCounter::add_batch` |
| `sync` post: __this_cpu_read(fbc.counters) == 0 ∧ sum unchanged | `PercpuCounter::sync` |
| `sum` post: returns fbc.count + Σ_{cpu ∈ online∪dying} shard[cpu] | `PercpuCounter::sum` |
| `compare` post: returns Greater iff sum > rhs (precise when borderline) | `PercpuCounter::compare` |
| `limited_add` post: good ⟹ sum after ≤ limit (amount>0) ∨ ≥ limit (amount<0); !good ⟹ sum unchanged | `PercpuCounter::limited_add` |
| `cpu_dead` post: per_cpu_ptr(fbc.counters, cpu) == 0 ∧ sum unchanged | `PercpuCounter::cpu_dead` |
| `compute_batch` post: PERCPU_COUNTER_BATCH == max(32, 2 * num_online_cpus) | `PercpuCounter::compute_batch` |

### Layer 4: Verus/Creusot functional

`PercpuCounter` semantic equivalence: for any sequence of operations [init(amount), op1, op2, …, opn] where opi ∈ {add(d), sub(d), set(v), sync(), destroy()}, the final precise sum equals the algebraic interpretation modulo destroy ordering. Approximate read bounded by `batch * num_online_cpus`. Per-Documentation/core-api/local_ops.rst (local cmpxchg semantics) and per-Documentation/admin-guide/cputopology.rst (cpu_online_mask / cpu_dying_mask).

## Hardening

(Inherits row-1 features from `lib/00-overview.md` § Hardening.)

PercpuCounter reinforcement:

- **Per-fbc->lock raw_spinlock** — defense against per-IRQ-context inversion (callers use it in softirq / hardirq).
- **Per-add_batch fast-path local-cmpxchg / IRQ-save** — defense against per-IRQ-mid-update double-fold.
- **Per-set zeros every *possible* CPU's shard** — defense against per-stale-shard-on-late-onlining.
- **Per-sum walks online ∪ dying** — defense against per-CPU-going-offline lost shard.
- **Per-destroy_many idempotent on null counters** — defense against per-double-destroy.
- **Per-CONFIG_DEBUG_OBJECTS_PERCPU_COUNTER fixup_free** — defense against per-free-while-active UAF.
- **Per-percpu_counters_lock for hotplug list** — defense against per-concurrent-init/destroy race.
- **Per-compute_batch_value at ONLINE** — defense against per-batch-undersize as CPUs come up (precision drifts wider).
- **Per-list_add at init / list_del at destroy** — defense against per-cpu_dead walking a stale fbc.
- **Per-limited_add precise-sum fallback at boundary** — defense against per-approximate-overshoot of limit (e.g., quota overrun).
- **Per-WARN_ON_ONCE(!fbc) in destroy** — defense against per-API-misuse with NULL pointer.

## Grsecurity/PaX-style Reinforcement

Baseline (apply to every Tier-3 surface):

- **PAX_USERCOPY** — counter snapshots returned via `/proc` (e.g. mm rss) traverse owner's slab whitelist on copy_to_user.
- **PAX_KERNEXEC** — no writable text; `percpu_counter_*` are pure data ops.
- **PAX_RANDKSTACK** — caller's syscall entry randomizes stack base.
- **PAX_REFCOUNT** — `percpu_counter.count` is an `s64`; PAX_REFCOUNT-style saturation applied for known-monotonic counters (e.g. inode count) via wrapping `refcount64_t` variant.
- **PAX_MEMORY_SANITIZE** — per-CPU shard array `__alloc_percpu` zero-on-alloc and zero-on-free.
- **PAX_UDEREF** — no user-pointer deref.
- **PAX_RAP / kCFI** — no indirect calls in core; subsystem callers' wrappers type-tagged independently.
- **GRKERNSEC_HIDESYM** — `__percpu_counter_*` symbols hidden from non-CAP_SYSLOG kallsyms.
- **GRKERNSEC_DMESG** — `WARN_ON_ONCE(!fbc)` gated behind CAP_SYSLOG.

Subsystem-specific reinforcement:

- **Per-CPU shard batch bound** — each shard is an `s32` clamped between `[-batch, +batch]` (typically `2 * num_online_cpus()`); shard overflow drains into the global `count` under `fbc->lock`, never wraps.
- **Limited-add precise-sum fallback** — `percpu_counter_limited_add` near the limit falls back to `percpu_counter_sum` (precise) rather than the approximate fast-path, defeating quota-overshoot attacks where an attacker contrives many CPUs to race past a limit using stale shard-only reads.
- **Destroy ordering** — `percpu_counter_destroy` `WARN_ON_ONCE(!fbc)` and removes from `percpu_counters` list under `percpu_counters_lock`; PaX additionally asserts `list_empty(&fbc->list)` invariant on every traversal.
- **Hotplug shard drain** — CPU-down handler drains the departing CPU's shard into the global counter atomically; PaX-REFCOUNT-style saturation applied to the global accumulator on drain.
- **Rationale** — percpu_counter underpins memcg/quota/rss/dirty accounting; its job is "fast and almost-right." The hardening makes "almost-right" never breach a hard limit while keeping the fast path lock-free, denying limit-bypass attacks.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `include/linux/percpu_counter.h` static-inline wrappers (`percpu_counter_init`, `_destroy`, `_inc`, `_dec`, `_add`, `_sub`, `_read`, `_read_positive`, `_sum_positive`) — trivial transformations of the underlying entry points; documented in this same file's API surface but no separate Tier-3 stub.
- `include/linux/local_lock.h`, `include/linux/this_cpu_ops.h` — primitives consumed (covered in `lib/data-structures.md` § per-CPU + atomic primitives, and arch-specific `arch/x86/00-overview.md` § per-CPU).
- `include/linux/fprop.h` and `lib/flex_proportions.c` — floating-proportions counter is a different (more complex) batched-counter primitive used by writeback; separate Tier-3.
- Specific callers (`mm/page-writeback.c`'s WB_*, `mm/util.c`'s `vm_committed_as`, `fs/dquot.c`'s `inuse_sem` counters, NFS counters) — covered in their respective Tier-3 docs.
- Implementation code.
