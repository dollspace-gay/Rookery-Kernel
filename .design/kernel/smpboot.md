# Tier-3: kernel/smpboot.c — SMP boot / per-CPU hotplug kthreads

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/smpboot.c (~322 lines)
  - kernel/smpboot.h
  - include/linux/smpboot.h
  - kernel/smp.c (smp_init caller)
  - kernel/cpu.c (cpuhp state machine caller)
-->

## Summary

`kernel/smpboot.c` provides two intertwined SMP-bring-up services. **(A) Per-CPU idle-thread storage**: `idle_thread_get(cpu)` returns the cached `swapper/<cpu>` task struct; `idle_thread_set_boot_cpu()` registers `current` as the boot CPU's idle on first call; `idle_threads_init()` pre-`fork_idle`s an idle task for every possible CPU at early-init so secondary CPUs have a runqueue task ready when they come online; `idle_init(cpu)` lazily forks an idle when the per-CPU slot is empty. **(B) Per-CPU hotplug-aware kthreads** (`struct smp_hotplug_thread` + `struct smpboot_thread_data`): subsystems (ksoftirqd, migration, cpuhp, watchdog, ...) register a descriptor describing per-CPU thread `setup`/`thread_fn`/`park`/`unpark`/`cleanup` callbacks; `smpboot_register_percpu_thread()` creates and unparks the threads on every online CPU and adds the descriptor to a global list so future hotplug events automatically create/park/unpark threads via `smpboot_create_threads()` / `_park_threads()` / `_unpark_threads()`. Each thread runs the shared `smpboot_thread_fn()` loop, which honours `kthread_should_stop`, `kthread_should_park`, and a `HP_THREAD_{NONE,ACTIVE,PARKED}` state machine to drive the registered callbacks at the correct lifecycle point. Critical for: deterministic CPU online/offline, percpu-affine work, runqueue idle availability, kexec/suspend coordination.

This Tier-3 covers `kernel/smpboot.c` (~322 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `idle_threads` (per-CPU `task_struct *`) | per-CPU swapper cache | `Smpboot::idle_threads` |
| `idle_thread_get(cpu)` | per-CPU idle accessor | `Smpboot::idle_thread_get` |
| `idle_thread_set_boot_cpu()` | per-boot-CPU register `current` as idle | `Smpboot::idle_thread_set_boot_cpu` |
| `idle_init(cpu)` (static inline) | per-CPU lazy `fork_idle` | `Smpboot::idle_init` |
| `idle_threads_init()` | per-init: fork all non-boot idles | `Smpboot::idle_threads_init` |
| `struct smp_hotplug_thread` | per-subsystem descriptor | `SmpHotplugThread` |
| `struct smpboot_thread_data` | per-(subsystem,CPU) runtime state | `SmpbootThreadData` |
| `enum {HP_THREAD_NONE, _ACTIVE, _PARKED}` | per-thread lifecycle state | `HpThreadState` |
| `smpboot_thread_fn(data)` | per-CPU kthread main loop | `Smpboot::thread_fn` |
| `__smpboot_create_thread(ht, cpu)` | per-(ht,CPU) create+park | `Smpboot::create_thread_inner` |
| `smpboot_create_threads(cpu)` | per-CPU-onlining: create all registered threads | `Smpboot::create_threads` |
| `smpboot_unpark_thread(ht, cpu)` | per-(ht,CPU) unpark | `Smpboot::unpark_thread` |
| `smpboot_unpark_threads(cpu)` | per-CPU-online: unpark all | `Smpboot::unpark_threads` |
| `smpboot_park_thread(ht, cpu)` | per-(ht,CPU) park | `Smpboot::park_thread` |
| `smpboot_park_threads(cpu)` | per-CPU-going-down: park all (reverse order) | `Smpboot::park_threads` |
| `smpboot_destroy_threads(ht)` | per-deregister: stop on all possible CPUs | `Smpboot::destroy_threads` |
| `smpboot_register_percpu_thread(plug)` | per-subsystem: register + start everywhere | `Smpboot::register_percpu_thread` |
| `smpboot_unregister_percpu_thread(plug)` | per-subsystem: stop + unlink | `Smpboot::unregister_percpu_thread` |
| `hotplug_threads` (list head) | per-system registry | `Smpboot::hotplug_threads` |
| `smpboot_threads_lock` (mutex) | per-list serialiser | `Smpboot::threads_lock` |

## Compatibility contract

REQ-1: Per-CPU `idle_threads` storage:
- `DEFINE_PER_CPU(struct task_struct *, idle_threads)`.
- Initial: NULL on every CPU.
- Gated by `CONFIG_GENERIC_SMP_IDLE_THREAD`.

REQ-2: `idle_thread_get(cpu)`:
- `tsk = per_cpu(idle_threads, cpu)`.
- if `!tsk`: return `ERR_PTR(-ENOMEM)`.
- return `tsk`.

REQ-3: `idle_thread_set_boot_cpu()` (`__init`):
- `per_cpu(idle_threads, smp_processor_id()) = current`.
- Caller is `start_kernel()` path before secondary bring-up.

REQ-4: `idle_init(cpu)` (static `__always_inline`):
- `tsk = per_cpu(idle_threads, cpu)`.
- if `tsk`: return (idempotent).
- `tsk = fork_idle(cpu)`.
- if `IS_ERR(tsk)`: `pr_err("SMP: fork_idle() failed for CPU %u\n", cpu)`; return.
- `per_cpu(idle_threads, cpu) = tsk`.

REQ-5: `idle_threads_init()` (`__init`):
- `boot_cpu = smp_processor_id()`.
- for each `cpu` in `for_each_possible_cpu`:
  - if `cpu != boot_cpu`: `idle_init(cpu)`.
- Post: every possible non-boot CPU has a forked idle ready.

REQ-6: `struct smp_hotplug_thread` (from `include/linux/smpboot.h`, owned by caller):
- `store`: `struct task_struct * __percpu *` — per-CPU task slot.
- `list`: `struct list_head` — registry link.
- `thread_should_run(cpu) -> bool` — work-available predicate.
- `thread_fn(cpu)` — per-iteration body.
- `thread_comm` — `kthread` name template (`"ksoftirqd/%u"`, ...).
- `setup(cpu)` — optional first-time setup.
- `cleanup(cpu, online)` — optional teardown.
- `park(cpu)` / `unpark(cpu)` — optional park hooks.
- `selfparking: bool` — true when `thread_fn` parks itself (no external `kthread_park`).

REQ-7: `struct smpboot_thread_data` (per-(ht,CPU) heap allocation, owned by us):
- `cpu: unsigned int`.
- `status: enum {HP_THREAD_NONE, _ACTIVE, _PARKED}`.
- `ht: struct smp_hotplug_thread *` — back-pointer.

REQ-8: `smpboot_thread_fn(data)` (main loop, returns 1 on exit, 0 otherwise — actually returns 0 on stop per source):
- `td = data; ht = td->ht`.
- Loop forever:
  - `set_current_state(TASK_INTERRUPTIBLE)`.
  - `preempt_disable()`.
  - **Stop path**: if `kthread_should_stop()`:
    - `__set_current_state(TASK_RUNNING); preempt_enable()`.
    - if `ht->cleanup && td->status != HP_THREAD_NONE`: `ht->cleanup(td->cpu, cpu_online(td->cpu))`.
    - `kfree(td)`; return 0.
  - **Park path**: if `kthread_should_park()`:
    - `__set_current_state(TASK_RUNNING); preempt_enable()`.
    - if `ht->park && td->status == HP_THREAD_ACTIVE`:
      - `BUG_ON(td->cpu != smp_processor_id())`.
      - `ht->park(td->cpu)`; `td->status = HP_THREAD_PARKED`.
    - `kthread_parkme()`; continue.
  - **CPU-affinity invariant**: `BUG_ON(td->cpu != smp_processor_id())`.
  - **Setup / unpark transitions** by `td->status`:
    - `HP_THREAD_NONE` → `__set_current_state(RUNNING); preempt_enable()`; if `ht->setup`: `ht->setup(td->cpu)`; `td->status = HP_THREAD_ACTIVE`; continue.
    - `HP_THREAD_PARKED` → `__set_current_state(RUNNING); preempt_enable()`; if `ht->unpark`: `ht->unpark(td->cpu)`; `td->status = HP_THREAD_ACTIVE`; continue.
  - **Work cycle**:
    - if `!ht->thread_should_run(td->cpu)`: `preempt_enable_no_resched(); schedule()`.
    - else: `__set_current_state(RUNNING); preempt_enable(); ht->thread_fn(td->cpu)`.

REQ-9: `__smpboot_create_thread(ht, cpu)`:
- `tsk = *per_cpu_ptr(ht->store, cpu)`.
- if `tsk`: return 0 (already exists; idempotent for hotplug re-online).
- `td = kzalloc_node(sizeof(*td), GFP_KERNEL, cpu_to_node(cpu))`.
- if `!td`: return `-ENOMEM`.
- `td->cpu = cpu; td->ht = ht`.
- `tsk = kthread_create_on_cpu(smpboot_thread_fn, td, cpu, ht->thread_comm)`.
- if `IS_ERR(tsk)`: `kfree(td)`; return `PTR_ERR(tsk)`.
- `kthread_set_per_cpu(tsk, cpu)`.
- `kthread_park(tsk)` — park before storing so it's quiescent until released.
- `get_task_struct(tsk)`; `*per_cpu_ptr(ht->store, cpu) = tsk`.
- if `ht->create`:
  - `if (!wait_task_inactive(tsk, TASK_PARKED)) WARN_ON(1);`
  - else `ht->create(cpu)`.
- return 0.

REQ-10: `smpboot_create_threads(cpu)`:
- `mutex_lock(&smpboot_threads_lock)`.
- for each `cur` in `hotplug_threads`: `ret = __smpboot_create_thread(cur, cpu)`; break on error.
- `mutex_unlock`; return `ret`.
- Caller: CPU hotplug state machine on CPU coming up.

REQ-11: `smpboot_unpark_thread(ht, cpu)`:
- `tsk = *per_cpu_ptr(ht->store, cpu)`.
- if `!ht->selfparking`: `kthread_unpark(tsk)`.

REQ-12: `smpboot_unpark_threads(cpu)`:
- `mutex_lock`; for each ht in `hotplug_threads` (forward): `smpboot_unpark_thread(ht, cpu)`; `mutex_unlock`.

REQ-13: `smpboot_park_thread(ht, cpu)`:
- `tsk = *per_cpu_ptr(ht->store, cpu)`.
- if `tsk && !ht->selfparking`: `kthread_park(tsk)`.

REQ-14: `smpboot_park_threads(cpu)`:
- `mutex_lock`; for each ht in `hotplug_threads` **reverse**: `smpboot_park_thread(ht, cpu)`; `mutex_unlock`.
- Reverse order mirrors setup order on the way down.

REQ-15: `smpboot_destroy_threads(ht)`:
- for each `cpu` in `for_each_possible_cpu`:
  - `tsk = *per_cpu_ptr(ht->store, cpu)`.
  - if `tsk`: `kthread_stop_put(tsk)`; `*per_cpu_ptr(ht->store, cpu) = NULL`.
- Note: includes parked threads of currently-offline CPUs.

REQ-16: `smpboot_register_percpu_thread(plug_thread)`:
- `cpus_read_lock()` — pin CPU mask.
- `mutex_lock(&smpboot_threads_lock)`.
- for each `cpu` in `for_each_online_cpu`:
  - `ret = __smpboot_create_thread(plug_thread, cpu)`.
  - if `ret`: `smpboot_destroy_threads(plug_thread)`; goto out.
  - `smpboot_unpark_thread(plug_thread, cpu)`.
- `list_add(&plug_thread->list, &hotplug_threads)`.
- out: `mutex_unlock`; `cpus_read_unlock`; return `ret`.
- `EXPORT_SYMBOL_GPL`.

REQ-17: `smpboot_unregister_percpu_thread(plug_thread)`:
- `cpus_read_lock()`.
- `mutex_lock(&smpboot_threads_lock)`.
- `list_del(&plug_thread->list)`.
- `smpboot_destroy_threads(plug_thread)`.
- `mutex_unlock`; `cpus_read_unlock`.
- `EXPORT_SYMBOL_GPL`.

REQ-18: Locking discipline:
- `smpboot_threads_lock` (mutex) serialises all mutations of `hotplug_threads` and per-CPU `store` writes.
- `cpus_read_lock`/`_unlock` brackets the (un)register paths to pin the online mask.
- All thread creation is bracketed inside both locks; thread runs after locks dropped.

## Acceptance Criteria

- [ ] AC-1: `idle_thread_get(cpu)` before any init: returns `ERR_PTR(-ENOMEM)`; after `idle_threads_init`: returns non-NULL for every possible CPU.
- [ ] AC-2: `idle_thread_set_boot_cpu()` exactly once at boot: per-CPU slot of `smp_processor_id()` == `current`.
- [ ] AC-3: `idle_init(cpu)` called twice on same CPU: second call is a no-op (no second `fork_idle`).
- [ ] AC-4: `idle_threads_init()` skips the boot CPU (boot CPU's idle is `current`, not forked).
- [ ] AC-5: `smpboot_thread_fn` honours `kthread_should_stop`: calls `cleanup(cpu, online)` iff `status != HP_THREAD_NONE`, frees `td`, returns 0.
- [ ] AC-6: `smpboot_thread_fn` honours `kthread_should_park`: calls `park(cpu)` iff `status == HP_THREAD_ACTIVE`, transitions to `HP_THREAD_PARKED`, blocks in `kthread_parkme`.
- [ ] AC-7: Per-CPU affinity invariant: `BUG_ON(td->cpu != smp_processor_id())` in non-stop, non-park paths.
- [ ] AC-8: `HP_THREAD_NONE` → `setup(cpu)` called once → `HP_THREAD_ACTIVE`.
- [ ] AC-9: `HP_THREAD_PARKED` → `unpark(cpu)` called once → `HP_THREAD_ACTIVE`.
- [ ] AC-10: `__smpboot_create_thread` on already-populated slot: returns 0 without alloc.
- [ ] AC-11: `__smpboot_create_thread` post: thread created, parked, `kthread_set_per_cpu`, `get_task_struct` taken, stored, `create(cpu)` invoked iff present and `wait_task_inactive(TASK_PARKED)` succeeded.
- [ ] AC-12: `smpboot_register_percpu_thread`: creates and unparks on every online CPU; adds to `hotplug_threads`; on alloc/create failure rolls back via `smpboot_destroy_threads`.
- [ ] AC-13: `smpboot_park_threads` traverses `hotplug_threads` in reverse order; `smpboot_unpark_threads` in forward order.
- [ ] AC-14: `selfparking==true`: external `_unpark_thread`/`_park_thread` skip the kthread call.
- [ ] AC-15: `smpboot_unregister_percpu_thread`: list_del + destroy_threads (including parked offline-CPU threads).

## Architecture

```
const CONFIG_GENERIC_SMP_IDLE_THREAD: bool;

#[per_cpu]
static IDLE_THREADS: PerCpu<Option<*TaskStruct>>;

static HOTPLUG_THREADS: Mutex<LinkedList<SmpHotplugThread>>;
static THREADS_LOCK: Mutex<()>;            // implicit via Mutex above

#[repr(u8)]
enum HpThreadState { None = 0, Active, Parked }

struct SmpHotplugThread {
  store: *PerCpu<*TaskStruct>,
  list: ListHead,
  thread_should_run: fn(cpu: u32) -> bool,
  thread_fn: fn(cpu: u32),
  thread_comm: &'static str,
  setup: Option<fn(cpu: u32)>,
  cleanup: Option<fn(cpu: u32, online: bool)>,
  park: Option<fn(cpu: u32)>,
  unpark: Option<fn(cpu: u32)>,
  create: Option<fn(cpu: u32)>,
  selfparking: bool,
}

struct SmpbootThreadData {
  cpu: u32,
  status: HpThreadState,
  ht: *SmpHotplugThread,
}
```

`Smpboot::idle_thread_get(cpu) -> Result<*TaskStruct, Errno>`:
1. tsk = IDLE_THREADS[cpu].
2. if tsk.is_none(): return Err(-ENOMEM).
3. return Ok(tsk.unwrap()).

`Smpboot::idle_thread_set_boot_cpu()`:
1. IDLE_THREADS[smp_processor_id()] = Some(current).

`Smpboot::idle_init(cpu)`:
1. if IDLE_THREADS[cpu].is_some(): return.
2. tsk = fork_idle(cpu).
3. if tsk.is_err(): pr_err!("SMP: fork_idle() failed for CPU {}", cpu); return.
4. IDLE_THREADS[cpu] = Some(tsk.unwrap()).

`Smpboot::idle_threads_init()`:
1. boot = smp_processor_id().
2. for cpu in for_each_possible_cpu():
   - if cpu != boot: Smpboot::idle_init(cpu).

`Smpboot::thread_fn(data: *SmpbootThreadData) -> i32`:
1. td = data; ht = td.ht.
2. loop:
   - set_current_state(TASK_INTERRUPTIBLE).
   - preempt_disable().
   - if kthread_should_stop():
     - __set_current_state(TASK_RUNNING); preempt_enable().
     - if ht.cleanup.is_some() && td.status != HP_THREAD_NONE: (ht.cleanup)(td.cpu, cpu_online(td.cpu)).
     - kfree(td).
     - return 0.
   - if kthread_should_park():
     - __set_current_state(TASK_RUNNING); preempt_enable().
     - if ht.park.is_some() && td.status == HP_THREAD_ACTIVE:
       - BUG_ON(td.cpu != smp_processor_id()).
       - (ht.park)(td.cpu); td.status = HP_THREAD_PARKED.
     - kthread_parkme(); continue.
   - BUG_ON(td.cpu != smp_processor_id()).
   - match td.status:
     - HP_THREAD_NONE: __set_current_state(RUNNING); preempt_enable(); if ht.setup.is_some(): (ht.setup)(td.cpu); td.status = HP_THREAD_ACTIVE; continue.
     - HP_THREAD_PARKED: __set_current_state(RUNNING); preempt_enable(); if ht.unpark.is_some(): (ht.unpark)(td.cpu); td.status = HP_THREAD_ACTIVE; continue.
     - HP_THREAD_ACTIVE: fallthrough.
   - if !(ht.thread_should_run)(td.cpu):
     - preempt_enable_no_resched(); schedule().
   - else:
     - __set_current_state(RUNNING); preempt_enable(); (ht.thread_fn)(td.cpu).

`Smpboot::create_thread_inner(ht, cpu) -> Result<(), Errno>`:
1. if (*ht.store)[cpu].is_some(): return Ok(()).
2. td = kzalloc_node::<SmpbootThreadData>(GFP_KERNEL, cpu_to_node(cpu))?.
3. td.cpu = cpu; td.ht = ht.
4. tsk = kthread_create_on_cpu(Smpboot::thread_fn, td, cpu, ht.thread_comm).
5. if tsk.is_err(): kfree(td); return Err(tsk.err()).
6. kthread_set_per_cpu(tsk, cpu).
7. kthread_park(tsk).
8. get_task_struct(tsk).
9. (*ht.store)[cpu] = Some(tsk).
10. if ht.create.is_some():
    - if !wait_task_inactive(tsk, TASK_PARKED): WARN_ON(1).
    - else: (ht.create)(cpu).
11. Ok(()).

`Smpboot::create_threads(cpu) -> Result<(), Errno>`:
1. THREADS_LOCK.lock().
2. for ht in HOTPLUG_THREADS:
   - r = Smpboot::create_thread_inner(ht, cpu).
   - if r.is_err(): break.
3. unlock; return r.

`Smpboot::park_threads(cpu)`:
1. THREADS_LOCK.lock().
2. for ht in HOTPLUG_THREADS.iter().rev():
   - tsk = (*ht.store)[cpu].
   - if tsk.is_some() && !ht.selfparking: kthread_park(tsk).
3. unlock.

`Smpboot::unpark_threads(cpu)`:
1. THREADS_LOCK.lock().
2. for ht in HOTPLUG_THREADS:
   - tsk = (*ht.store)[cpu].
   - if !ht.selfparking: kthread_unpark(tsk).
3. unlock.

`Smpboot::destroy_threads(ht)`:
1. for cpu in for_each_possible_cpu():
   - tsk = (*ht.store)[cpu].
   - if tsk.is_some():
     - kthread_stop_put(tsk); (*ht.store)[cpu] = None.

`Smpboot::register_percpu_thread(plug) -> Result<(), Errno>`:
1. cpus_read_lock().
2. THREADS_LOCK.lock().
3. for cpu in for_each_online_cpu():
   - r = Smpboot::create_thread_inner(plug, cpu).
   - if r.is_err():
     - Smpboot::destroy_threads(plug).
     - goto out.
   - Smpboot::unpark_thread(plug, cpu).
4. HOTPLUG_THREADS.push_back(plug).
5. out: unlock; cpus_read_unlock(); return r.

`Smpboot::unregister_percpu_thread(plug)`:
1. cpus_read_lock().
2. THREADS_LOCK.lock().
3. HOTPLUG_THREADS.remove(plug).
4. Smpboot::destroy_threads(plug).
5. unlock; cpus_read_unlock().

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `idle_get_uninit_returns_enomem` | INVARIANT | per-idle_thread_get: tsk == None ⟹ Err(-ENOMEM). |
| `idle_init_idempotent` | INVARIANT | per-idle_init: second call when slot occupied does nothing. |
| `idle_threads_init_skips_boot` | INVARIANT | per-idle_threads_init: boot CPU slot unchanged. |
| `thread_fn_cpu_affinity` | INVARIANT | per-smpboot_thread_fn: in non-stop/non-park paths, td.cpu == smp_processor_id(). |
| `thread_fn_status_progression` | INVARIANT | per-smpboot_thread_fn: NONE → ACTIVE (after setup); PARKED → ACTIVE (after unpark). |
| `cleanup_only_after_setup` | INVARIANT | per-smpboot_thread_fn-stop: cleanup invoked iff status != HP_THREAD_NONE. |
| `create_inner_alloc_failure_no_leak` | INVARIANT | per-create_thread_inner: kthread_create failure ⟹ td kfree'd. |
| `create_inner_thread_parked_on_return` | INVARIANT | per-create_thread_inner: kthread_park called before store. |
| `register_rollback_on_failure` | INVARIANT | per-register_percpu_thread: any error ⟹ destroy_threads runs before unlock. |
| `park_order_reverse_of_unpark` | INVARIANT | per-park_threads vs unpark_threads: traversal orders are reverse. |
| `threads_lock_held_during_list_mutation` | INVARIANT | per-list_add / list_del: THREADS_LOCK held. |
| `cpus_read_lock_brackets_register` | INVARIANT | per-register/unregister: cpus_read_lock held across body. |

### Layer 2: TLA+

`kernel/smpboot.tla`:
- States: per-CPU `idle_thread_slot ∈ {None, Some(tsk)}`; per-(ht,cpu) `state ∈ {Absent, Created_Parked, Active, Parked, Stopped}`.
- Actions: `IdleThreadSetBootCpu`, `IdleInit(cpu)`, `IdleThreadsInit`, `RegisterPercpuThread(ht)`, `UnregisterPercpuThread(ht)`, `CpuOnline(cpu)` (calls `CreateThreads + UnparkThreads`), `CpuOffline(cpu)` (calls `ParkThreads`), `ThreadFnIter(ht, cpu)` (covers setup/unpark/work/park/stop).
- Properties:
  - `safety_idle_get_after_init_some` — per-CPU: after `IdleThreadsInit`, `idle_thread_slot[cpu] != None` for all possible cpu.
  - `safety_active_on_correct_cpu` — per-(ht,cpu): when in `Active`, current CPU == cpu (per BUG_ON invariant).
  - `safety_park_before_offline` — per-CPU offline transition: every registered ht for that cpu reaches `Parked` (forward direction in TLA: action `ParkThreads` precedes hardware offline).
  - `safety_reverse_park_order` — per-cpu-offline: ht iteration order is the reverse of register order.
  - `safety_no_dangling_td` — per-thread stop: `td` heap pointer not reachable post-stop (kfree'd).
  - `liveness_register_eventually_active` — per-register_percpu_thread: every online cpu eventually reaches `Active` (assuming setup callback terminates).
  - `liveness_unregister_eventually_clean` — per-unregister: every per-CPU store slot is `None` post-call.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Smpboot::idle_thread_get` post: `Ok(_) ⟹ slot.is_some()`; `Err(-ENOMEM) ⟹ slot.is_none()` | `Smpboot::idle_thread_get` |
| `Smpboot::idle_init` post: `slot.is_some()` (assuming fork_idle ok) | `Smpboot::idle_init` |
| `Smpboot::idle_threads_init` post: ∀cpu ∈ possible \ {boot}: `slot.is_some()` | `Smpboot::idle_threads_init` |
| `Smpboot::thread_fn` invariant: `td.status` transitions ⊆ {None→Active, Parked→Active, Active→Parked} | `Smpboot::thread_fn` |
| `Smpboot::create_thread_inner` post: `Ok(_) ⟹ (*ht.store)[cpu].is_some() ∧ tsk parked`; `Err(_) ⟹ td freed` | `Smpboot::create_thread_inner` |
| `Smpboot::register_percpu_thread` post: `Ok(_) ⟹ plug ∈ HOTPLUG_THREADS ∧ ∀online cpu: store[cpu].is_some()` | `Smpboot::register_percpu_thread` |
| `Smpboot::unregister_percpu_thread` post: `plug ∉ HOTPLUG_THREADS ∧ ∀possible cpu: store[cpu].is_none()` | `Smpboot::unregister_percpu_thread` |
| `Smpboot::park_threads` post: ∀ht ∈ HOTPLUG_THREADS: store[cpu] parked (modulo `selfparking`) | `Smpboot::park_threads` |

### Layer 4: Verus/Creusot functional

`Per-register_percpu_thread → for_each_online_cpu(create_thread_inner ; unpark_thread) → list_add` ≡ Linux upstream call sequence per `kernel/smpboot.c:284-305`.

`Per-CPU hotplug bring-up: smpboot_create_threads(cpu) → smpboot_unpark_threads(cpu) ; bring-down: smpboot_park_threads(cpu) [reverse]` ≡ `kernel/cpu.c` CPUHP state machine slots `CPUHP_AP_SMPBOOT_THREADS` (up) / `CPUHP_AP_SMPBOOT_THREADS` (down).

`Per-thread loop: TASK_INTERRUPTIBLE → preempt_disable → {stop|park|setup|unpark|work|sleep}` semantic equivalence per Documentation/core-api/cpu_hotplug.rst and `include/linux/smpboot.h`.

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

SMP-boot reinforcement:

- **Per-CPU affinity BUG_ON in thread_fn** — defense against per-runaway-kthread executing off-CPU after a migration bug.
- **Per-kthread_park-before-store** — defense against per-race where a not-yet-parked thread runs before `setup` is wired up.
- **Per-status state machine (NONE → ACTIVE → PARKED)** — defense against per-double-setup or per-cleanup-without-setup.
- **Per-cleanup gated on `status != HP_THREAD_NONE`** — defense against per-NULL-deref in subsystem cleanup callback that assumes setup ran.
- **Per-`wait_task_inactive(TASK_PARKED)` before `create()`** — defense against per-runqueue-race where `create()` observes a still-running task.
- **Per-reverse-order park on offline** — defense against per-teardown-order violation across stacked subsystems.
- **Per-`cpus_read_lock` bracket on register/unregister** — defense against per-concurrent CPU hotplug racing list mutation.
- **Per-`smpboot_threads_lock` mutex** — defense against per-concurrent register/unregister corrupting `hotplug_threads`.
- **Per-`get_task_struct` on store** — defense against per-task-struct UAF if subsystem holds the pointer.
- **Per-`kthread_stop_put` on destroy** — defense against per-task ref leak across deregister.
- **Per-rollback on register failure (`smpboot_destroy_threads`)** — defense against per-partial-registration leak.
- **Per-`selfparking` honoured by external park/unpark** — defense against per-double-park deadlock with self-parking threads (e.g., migration).
- **Per-`fork_idle` failure logged and slot left NULL** — defense against per-silent-fail bringing up CPU with no idle.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — smpboot kthread parameters and sysfs CPU hotplug controls bounds-checked; no `smpboot_thread_data` slab leakage.
- **PAX_KERNEXEC** — `smp_init`, `secondary_startup_64`, and the AP-bringup trampoline reside in W^X kernel text; the trampoline page never becomes RW+X concurrently.
- **PAX_RANDKSTACK** — per-syscall kstack offset extended to per-AP-bringup idle kstack so secondary CPU stacks are not at predictable offsets.
- **PAX_REFCOUNT** — `smpboot_threads.refcount`, `smp_hotplug_thread` registration counts, and `per_cpu(cpu_online_mask)` accounting saturating-refcounted.
- **PAX_MEMORY_SANITIZE** — freed `task_struct` of decommissioned idle threads scrubbed at hotplug-down so a future bringup cannot inherit stale state.
- **PAX_UDEREF** — AP bringup paths dereference `cpu_init` / per-CPU data via kernel mappings only; user mappings unreachable until the AP joins `init_mm`'s normal context.
- **PAX_RAP / kCFI** — `smp_hotplug_thread` op callbacks (`thread_fn`, `setup`, `park`, `unpark`, `cleanup`) type-signatured; mismatched signature is a hard CFI fault.
- **GRKERNSEC_HIDESYM** — `smpboot_threads`, AP trampoline addresses, and `cpu_present_mask` hidden from `/proc/kallsyms`.
- **GRKERNSEC_DMESG** — AP bringup failure splats (`smpboot: CPU N failed to come online`) gated to CAP_SYSLOG.
- **AP bringup PAX_KERNEXEC** — secondary startup vector (`trampoline_start64`, `secondary_startup_64`) lives in `.text`; the trampoline page is RX-only at the moment of `cpu_up()`.
- **Trampoline page W^X** — real-mode trampoline allocated below 1 MiB is mapped RX during bringup and either reclaimed or made RO after `cpu_online`; never RW+X simultaneously, defeating "AP-bringup-as-W^X-bypass" tricks.
- **Per-CPU thread isolation** — each `smp_hotplug_thread` has its own per-CPU `task_struct` and `thread_fn`; PAX_REFCOUNT prevents an attacker-controlled register/unregister race from leaking a kthread into another CPU's slot.
- **Rationale** — AP bringup is the only path where the kernel is single-threaded yet executing a trampoline at a chosen physical address; corrupted trampoline or per-CPU idle creates a permanent kernel-mode foothold that survives every later defence. Grsec/PaX hardening keeps the bringup window an RX-only, refcounted, CFI-checked transition.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `kernel/cpu.c` CPUHP state machine driver (covered separately if expanded)
- `kernel/smp.c` `smp_init`, `smp_call_function`, IPI plumbing (covered separately)
- `kernel/kthread.c` (`kthread_create_on_cpu`, `kthread_park`, `kthread_should_park`) — covered by Tier-3 `kthread.md` if expanded
- `kernel/fork.c::fork_idle` (covered in `fork.md` Tier-3)
- Architecture-specific secondary CPU startup trampoline (`arch/*/kernel/smpboot.c`)
- Specific per-CPU thread subsystems (ksoftirqd, migration, cpuhp threads, watchdog) — each covered by their own Tier-3 if expanded
- `include/linux/smpboot.h` field semantics beyond what is referenced here
- Implementation code
