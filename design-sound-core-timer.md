---
title: "Tier-3: sound/core/timer.c — ALSA timer abstract layer"
tags: ["tier-3", "sound", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The **ALSA timer abstract layer** provides a kernel-wide tick service that PCM, sequencer, and userspace clients all share: a `struct snd_timer` (registered backend — system jiffies, hrtimer, card-side, or PCM-derived) drives `struct snd_timer_instance` consumers via tick-aligned callbacks. Per-`snd_timer_new` registers a backend with a `struct snd_timer_hardware` ops vector (`open` / `close` / `start` / `stop` / `set_period` / `c_resolution` / `precise_resolution`). Per-`snd_timer_open` binds a per-instance owner to a backend (master) or to a slave class (PCM, sequencer-OSS). Per-`snd_timer_start` / `_stop` / `_continue` / `_pause` transitions arm tick generation. Per-`snd_timer_interrupt` (called from backend ISR / hrtimer / system-timer expiry) walks the active-list, decrements `cticks`, sets / clears `RUNNING`, and enqueues to fast `ack_list` (in-IRQ callbacks) or slow `sack_list` (`system_highpri_wq` work item). Per-`/dev/snd/timer` char-dev (minor `SNDRV_MINOR_TIMER`) exposes `snd_timer_user` (ring queue of `snd_timer_read` / `snd_timer_tread64` records) for userspace pollers. Per-`SNDRV_TIMER_HW_SLAVE` allows OSS-sequencer / sequencer-side slave instances linked under a master tick. Critical for: PCM tick-driven playback timing, sequencer event scheduling, JACK-class low-latency userspace audio.

This Tier-3 covers `sound/core/timer.c` (~2526 lines).

### Acceptance Criteria

- [ ] AC-1: `snd_timer_new` + `_global_register` registers a backend; appears at `/proc/asound/timers`.
- [ ] AC-2: `snd_timer_open(SLAVE_CLASS, sclass=OSS_SEQUENCER)` parks the instance on slave-list; matches when master with same slave_id opens.
- [ ] AC-3: `snd_timer_open(GLOBAL_SYSTEM)` succeeds; `snd_timer_start(ticks)` arms with sticks = ticks; hw.start invoked.
- [ ] AC-4: Period of `(resolution × ticks) < 100 µs` returns -EINVAL.
- [ ] AC-5: `snd_timer_interrupt` decrements cticks; expired non-AUTO clears RUNNING; AUTO reloads cticks = ticks.
- [ ] AC-6: Fast callbacks (`IFLG_FAST` or `HW_WORK`) execute under timer->lock relax-reacquire; slow callbacks deferred to `system_highpri_wq`.
- [ ] AC-7: `snd_timer_pause` preserves cticks; `_continue` resumes only from IFLG_PAUSED (else EINVAL).
- [ ] AC-8: `snd_timer_stop` of last running instance invokes hw.stop; subsequent start invokes hw.start.
- [ ] AC-9: `/dev/snd/timer` read blocks on qchange_sleep until enqueue; coalesces adjacent same-resolution `snd_timer_read` entries.
- [ ] AC-10: TREAD_FORMAT_TIME64 emits `snd_timer_tread64{event, tstamp_sec, tstamp_nsec, val}`; FORMAT_TIME32 truncates correctly.
- [ ] AC-11: Card shutdown → `snd_timer_user_disconnect` → poll returns EPOLLERR; read returns -ENODEV.
- [ ] AC-12: max_instances = 1000 enforced; 1001st open returns -EBUSY.
- [ ] AC-13: System-timer correction tracks `(jiffies - last_expires)` to absorb late-tick jitter.
- [ ] AC-14: `snd_timer_close` blocks while IFLG_CALLBACK set (busy-wait udelay(10)).
- [ ] AC-15: hrtimer backend `SNDRV_TIMER_GLOBAL_HRTIMER` registered when `CONFIG_SND_HRTIMER=y`.

### Architecture

```
struct SndTimer {
  tmr_class: u8,                        // SNDRV_TIMER_CLASS_*
  card: Option<*SndCard>,
  module: Option<*Module>,
  tmr_device: i32,
  tmr_subdevice: i32,
  id: [u8; 64],
  name: [u8; 80],
  flags: AtomicU32,                     // SNDRV_TIMER_FLG_*
  running: i32,
  sticks: u64,
  private_data: Option<NonNull<()>>,
  private_free: Option<fn(&mut SndTimer)>,
  hw: SndTimerHardware,
  lock: SpinLock<()>,
  device_list: ListHead,
  open_list_head: ListHead,
  active_list_head: ListHead,
  ack_list_head: ListHead,
  sack_list_head: ListHead,
  task_work: WorkStruct,
  max_instances: i32,
  num_instances: i32,
}

struct SndTimerHardware {
  flags: u32,                           // SNDRV_TIMER_HW_*
  resolution: u64,
  resolution_min: u64,
  resolution_max: u64,
  ticks: u64,
  open: Option<fn(&mut SndTimer) -> i32>,
  close: Option<fn(&mut SndTimer) -> i32>,
  c_resolution: Option<fn(&SndTimer) -> u64>,
  start: fn(&mut SndTimer) -> i32,
  stop: fn(&mut SndTimer) -> i32,
  set_period: Option<fn(&mut SndTimer, u64, u64) -> i32>,
  precise_resolution: Option<fn(&SndTimer, &mut u64, &mut u64) -> i32>,
}

struct SndTimerInstance {
  timer: Option<*SndTimer>,
  owner: CString,
  flags: u32,                           // SNDRV_TIMER_IFLG_*
  private_data: Option<NonNull<()>>,
  private_free: Option<fn(&mut SndTimerInstance)>,
  callback: Option<fn(&SndTimerInstance, u64, u64)>,
  ccallback: Option<fn(&SndTimerInstance, i32, &Timespec64, u64)>,
  disconnect: Option<fn(&SndTimerInstance)>,
  callback_data: Option<NonNull<()>>,
  ticks: u64,
  cticks: u64,
  pticks: u64,
  resolution: u64,
  lost: u64,
  slave_class: i32,
  slave_id: u32,
  open_list: ListHead,
  active_list: ListHead,
  master_list: ListHead,
  ack_list: ListHead,
  slave_list_head: ListHead,
  slave_active_head: ListHead,
  master: Option<*SndTimerInstance>,
}

struct SndTimerUser {
  timeri: Option<*SndTimerInstance>,
  tread: TreadFormat,                   // NONE | TIME32 | TIME64
  ticks: u64,
  overrun: u64,
  qhead: i32, qtail: i32, qused: i32, queue_size: i32,
  disconnected: bool,
  queue: Box<[SndTimerRead]>,
  tqueue: Box<[SndTimerTread64]>,
  qlock: SpinLock<()>,
  last_resolution: u64,
  filter: u32,
  tstamp: Timespec64,
  qchange_sleep: WaitQueueHead,
  fasync: Option<*SndFasync>,
  ioctl_lock: Mutex<()>,
}
```

`SndTimer::open(timeri, tid, slave_id) -> Result<()>`:
1. /* Lock list */
2. Hold REGISTER_MUTEX.
3. If tid.dev_class == SLAVE:
   - Validate dev_sclass ∈ (NONE, OSS_SEQUENCER].
   - EBUSY if num_slaves ≥ MAX_SLAVE_INSTANCES.
   - Stamp timeri.{slave_class, slave_id}; set IFLG_SLAVE.
   - list_add_tail SLAVE_LIST.
   - num_slaves += 1.
   - SndTimer::check_slave(timeri).
4. Else:
   - timer = SndTimer::find(tid).
   - /* Module autoload retry */
   - If !timer ∧ CONFIG_MODULES: drop mutex, request_module, reacquire, refind.
   - ENODEV if still missing.
   - EBUSY if open_list nonempty ∧ head.flags & IFLG_EXCLUSIVE.
   - EBUSY if num_instances ≥ max_instances.
   - try_module_get(timer.module) — EBUSY on fail.
   - If timer.card: get_device(&card.card_dev); record card_dev_to_put.
   - If open_list empty ∧ hw.open: invoke hw.open(timer); on err module_put + unwind.
   - timeri.timer = timer; timeri.slave_class = sclass; timeri.slave_id = sid.
   - list_add_tail open_list ⊳ timer.open_list_head.
   - If has_slave_key(timeri): list_add_tail master_list ⊳ MASTER_LIST.
   - timer.num_instances += 1.
   - SndTimer::check_master(timeri).
5. /* Unwind on error */
6. If err: close_locked(timeri, &card_dev_to_put).
7. Drop mutex.
8. /* put_device after mutex drop to avoid deadlock */
9. If err ∧ card_dev_to_put: put_device(...).

`SndTimer::interrupt(timer, ticks_left)`:
1. If timer.card.shutdown: clear_callbacks(ack_list_head); return.
2. Hold timer.lock.
3. resolution = c_resolution() ?: hw.resolution.
4. For ti in active_list_head (safe-iter):
   - skip IFLG_DEAD.
   - skip !IFLG_RUNNING.
   - ti.pticks += ticks_left.
   - ti.resolution = resolution.
   - ti.cticks = saturating_sub(ti.cticks, ticks_left).
   - if ti.cticks > 0: continue.
   - if IFLG_AUTO: ti.cticks = ti.ticks.
   - else: clear IFLG_RUNNING; timer.running -= 1; list_del_init active_list.
   - ack_head = if (hw.flags & HW_WORK) ∨ (ti.flags & IFLG_FAST) { &ack_list_head } else { &sack_list_head }.
   - list_add_tail ti.ack_list ⊳ ack_head.
   - For ts in ti.slave_active_head: ts.pticks = ti.pticks; ts.resolution = resolution; list_add_tail ts.ack_list.
5. If FLG_RESCHED: reschedule(timer, sticks).
6. If timer.running:
   - if HW_STOP: hw.stop(timer); set FLG_CHANGE.
   - if !HW_AUTO ∨ FLG_CHANGE: clear FLG_CHANGE; hw.start(timer).
7. Else: hw.stop(timer).
8. process_callbacks(timer, &ack_list_head). /* fast */
9. If !sack_list_head.empty(): queue_work(system_highpri_wq, &task_work).

`SndTimer::reschedule(timer, ticks_left)`:
1. ticks = !0u64.
2. For ti in active_list_head:
   - if IFLG_START: clear START, set RUNNING, timer.running += 1.
   - if IFLG_RUNNING: ticks = min(ticks, ti.cticks).
3. If ticks == !0: clear FLG_RESCHED; return.
4. ticks = min(ticks, hw.ticks).
5. If ticks ≠ ticks_left: set FLG_CHANGE.
6. timer.sticks = ticks.

`SndTimer::process_callbacks(timer, head)`:
1. While !head.empty():
   - ti = list_first_entry(head, ack_list).
   - list_del_init &ti.ack_list.
   - If !IFLG_DEAD:
     - ticks = ti.pticks; ti.pticks = 0; resolution = ti.resolution.
     - set IFLG_CALLBACK.
     - drop timer.lock; ti.callback(ti, resolution, ticks); reacquire timer.lock.
     - clear IFLG_CALLBACK.

`SndTimer::start1(timeri, start: bool, ticks: u64) -> i32`:
1. timer = timeri.timer or return -EINVAL.
2. Hold timer.lock.
3. EINVAL if IFLG_DEAD.
4. ENODEV if card.shutdown.
5. EBUSY if IFLG_RUNNING | IFLG_START.
6. If start ∧ !HW_SLAVE:
   - EINVAL if hw_resolution(timer) × ticks < 100_000.
7. If start: timeri.ticks = timeri.cticks = ticks.
8. Else if cticks == 0: cticks = 1.
9. timeri.pticks = 0.
10. list_move_tail active_list ⊳ timer.active_list_head.
11. If timer.running:
    - if HW_SLAVE: jump start_now.
    - set FLG_RESCHED | IFLG_START.
    - result = 1.
12. Else:
    - if start: timer.sticks = ticks.
    - hw.start(timer).
    - start_now: timer.running += 1; set IFLG_RUNNING; result = 0.
13. notify1(timeri, if start { EVENT_START } else { EVENT_CONTINUE }).
14. return result.

`SndTimerUser::interrupt(timeri, resolution, ticks)`:
1. tu = timeri.callback_data as &SndTimerUser.
2. Hold tu.qlock.
3. /* Coalesce */
4. If tu.qused > 0:
   - prev = (tu.qtail == 0) ? queue_size - 1 : tu.qtail - 1.
   - if tu.queue[prev].resolution == resolution: tu.queue[prev].ticks += ticks; goto wake.
5. If tu.qused ≥ queue_size: tu.overrun += 1.
6. Else: tu.queue[tu.qtail] = SndTimerRead{resolution, ticks}; tu.qtail = (tu.qtail + 1) % queue_size; tu.qused += 1.
7. wake: snd_kill_fasync(tu.fasync, SIGIO, POLL_IN); wake_up(&tu.qchange_sleep).

### Out of Scope

- sound/core/hrtimer.c hrtimer-specific backend (covered separately if expanded)
- sound/core/pcm_timer.c PCM-derived backend (covered in `pcm.md` Tier-3)
- sound/core/timer_compat.c 32-bit compat ioctls (covered with SYSCALL_COMPAT track)
- ALSA sequencer-side queue timer (`seq.md` Tier-3 — uses snd_timer_instance as consumer)
- include/uapi/sound/asound.h UAPI struct layouts (covered in `sound/uapi/asound.md` if expanded)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct snd_timer` | per-backend | `SndTimer` |
| `struct snd_timer_hardware` | per-backend ops vtable | `SndTimerHardware` |
| `struct snd_timer_instance` | per-consumer | `SndTimerInstance` |
| `struct snd_timer_user` | per-userspace fd state | `SndTimerUser` |
| `struct snd_timer_system_private` | per-system-timer (jiffies) state | `SndTimerSystemPrivate` |
| `struct snd_utimer` | per-userspace-driven timer state | `SndUserTimer` |
| `snd_timer_instance_new()` / `_free()` | per-alloc / per-free instance | `SndTimerInstance::new` / `drop` |
| `snd_timer_open()` | per-bind instance to backend | `SndTimer::open` |
| `snd_timer_close()` | per-unbind + drain | `SndTimer::close` |
| `snd_timer_start()` | per-arm (fresh ticks) | `SndTimer::start` |
| `snd_timer_stop()` | per-disarm + reset cticks | `SndTimer::stop` |
| `snd_timer_continue()` | per-resume from PAUSED | `SndTimer::continue_` |
| `snd_timer_pause()` | per-pause-keeping-position | `SndTimer::pause` |
| `snd_timer_resolution()` | per-query backend ns/tick | `SndTimer::resolution` |
| `snd_timer_interrupt()` | per-tick entry from backend | `SndTimer::interrupt` |
| `snd_timer_reschedule()` | per-recompute sticks across instances | `SndTimer::reschedule` |
| `snd_timer_process_callbacks()` | per-walk ack/sack list | `SndTimer::process_callbacks` |
| `snd_timer_work()` | per-deferred slow ack worker | `SndTimer::work` |
| `snd_timer_notify()` | per-slave-master event broadcast | `SndTimer::notify` |
| `snd_timer_notify1()` | per-instance event broadcast | `SndTimer::notify1` |
| `snd_timer_new()` | per-register backend | `SndTimer::new` |
| `snd_timer_free()` | per-unregister + drain | `SndTimer::free` |
| `snd_timer_global_new()` / `_register()` / `_free()` | per-global (non-card) backend | `SndTimer::global_*` |
| `snd_timer_register_system()` | per-jiffies-driven backend | `SndTimer::register_system` |
| `snd_timer_s_function()` / `_start()` / `_stop()` / `_close()` | per-system-timer ops | `SndTimer::system_ops` |
| `snd_timer_user_open()` / `_release()` / `_read()` / `_poll()` / `_ioctl()` | per-/dev/snd/timer fops | `SndTimerUser::file_ops` |
| `snd_timer_user_interrupt()` / `_ccallback()` / `_tinterrupt()` | per-callback into user queue | `SndTimerUser::interrupt` / `ccallback` / `tinterrupt` |
| `snd_utimer_open()` / `_start()` / `_stop()` / `_trigger()` / `_ioctl()` | per-userspace-driven timer | `SndUserTimer::*` |
| `register_mutex` | per-list serializer | `SndTimer::REGISTER_MUTEX` |
| `slave_active_lock` | per-slave-active-list spin | `SndTimer::SLAVE_ACTIVE_LOCK` |

### compatibility contract

REQ-1: struct snd_timer (per `include/sound/timer.h`):
- tmr_class: SNDRV_TIMER_CLASS_NONE / _SLAVE / _GLOBAL / _CARD / _PCM.
- card: per-card parent (NULL for global).
- module: per-owner module for `try_module_get`.
- tmr_device / tmr_subdevice: per-backend identity tuple.
- id[64] / name[80]: per-backend strings.
- flags: SNDRV_TIMER_FLG_CHANGE / _RESCHED.
- running: per-active instance count.
- sticks: per-current schedule-tick countdown.
- hw: snd_timer_hardware ops + flags + resolution.
- lock: per-backend spinlock guarding active/ack lists + counters.
- device_list / open_list_head / active_list_head / ack_list_head / sack_list_head: per-list anchors.
- task_work: per-slow-ack work item.
- max_instances / num_instances: per-backend instance cap (default 1000).
- private_data / private_free: per-backend driver state.

REQ-2: struct snd_timer_hardware:
- flags: SNDRV_TIMER_HW_AUTO | _STOP | _SLAVE | _FIRST | _WORK.
- resolution: per-tick ns (average).
- resolution_min / _max: per-range bounds.
- ticks: per-interrupt max ticks.
- open / close: per-backend prepare / release.
- c_resolution: per-callback to query live resolution.
- start / stop: per-arm / disarm backend hardware.
- set_period: per-program period (num / den fraction).
- precise_resolution: per-query exact (num / den) for slave.

REQ-3: struct snd_timer_instance:
- timer: per-bound backend (NULL for unattached / slave-waiting).
- owner: per-kstrdup'd owner string (for /proc/asound/timers).
- flags: SNDRV_TIMER_IFLG_SLAVE | _RUNNING | _START | _AUTO | _FAST | _CALLBACK | _EXCLUSIVE | _EARLY_EVENT | _PAUSED (internal) | _DEAD (internal).
- private_data / private_free: per-consumer state.
- callback(timeri, ticks, resolution): per-tick fast / work callback.
- ccallback(timeri, event, tstamp, resolution): per-control event callback.
- disconnect(timeri): per-card-shutdown notification.
- callback_data: per-callback opaque.
- ticks / cticks / pticks: per-auto-load / countdown / accumulated.
- resolution: per-last-snapshot ns.
- lost: per-missed-tick counter.
- slave_class / slave_id: per-slave id tuple (matched to master).
- open_list / active_list / master_list / ack_list / slave_list_head / slave_active_head: per-list members.
- master: per-master-slave linkage (NULL for masters).

REQ-4: snd_timer_instance_new(owner):
- kzalloc snd_timer_instance.
- kstrdup owner string.
- INIT_LIST_HEAD on all 6 list heads (open / active / master / ack / slave_list_head / slave_active_head).
- return instance.

REQ-5: snd_timer_new(card, id, tid, &rtimer):
- /* Validate */
- WARN if tid->dev_class ∈ {CARD, PCM} ∧ card == NULL.
- kzalloc snd_timer + populate tmr_class / card / tmr_device / tmr_subdevice / id.
- sticks = 1; max_instances = 1000.
- spin_lock_init(&timer->lock).
- INIT_WORK(&timer->task_work, snd_timer_work).
- If card: snd_device_new(card, SNDRV_DEV_TIMER, timer, &ops) with `dev_register = snd_timer_dev_register`, `dev_free = snd_timer_dev_free`, `dev_disconnect = snd_timer_dev_disconnect`.
- *rtimer = timer.

REQ-6: snd_timer_dev_register(dev):
- BUG_ON !timer->hw.start ∨ !timer->hw.stop.
- Validate (hw.resolution ≠ 0 ∨ hw.c_resolution ≠ NULL) unless SNDRV_TIMER_HW_SLAVE.
- Hold register_mutex.
- Insert into snd_timer_list ordered by (tmr_class, card-number, tmr_device, tmr_subdevice); EBUSY on conflict.

REQ-7: snd_timer_open(timeri, tid, slave_id):
- Hold register_mutex.
- If tid->dev_class == SNDRV_TIMER_CLASS_SLAVE:
  - Validate dev_sclass ∈ (NONE, OSS_SEQUENCER].
  - EBUSY if num_slaves ≥ MAX_SLAVE_INSTANCES (1000).
  - Set IFLG_SLAVE; record slave_class / slave_id; list_add_tail open_list ⊳ snd_timer_slave_list; num_slaves++; snd_timer_check_slave().
- Else (master open):
  - snd_timer_find(tid); request_module on miss (CONFIG_MODULES); refetch.
  - ENODEV if not found.
  - EBUSY if first existing instance has IFLG_EXCLUSIVE.
  - EBUSY if num_instances ≥ max_instances.
  - try_module_get(timer->module) — EBUSY on fail.
  - If timer->card: get_device(&card->card_dev) for safe disconnect.
  - If open_list_head empty ∧ hw.open: invoke hw.open(timer); on error module_put + bail.
  - Set timeri->timer / slave_class / slave_id; list_add_tail open_list ⊳ timer->open_list_head; possibly list_add master_list; num_instances++; snd_timer_check_master().

REQ-8: snd_timer_close():
- Hold register_mutex via scoped_guard.
- Acquire timer->lock; set IFLG_DEAD.
- list_del_init open_list; if SLAVE: num_slaves--.
- snd_timer_stop(timeri).
- num_instances--.
- /* Wait for in-flight callback */
- while timeri->flags & IFLG_CALLBACK: udelay(10) under timer->lock cycling.
- remove_slave_links() — re-parent slaves back to snd_timer_slave_list.
- If !SLAVE ∧ open_list_head empty ∧ hw.close: invoke hw.close(timer).
- card_dev put_device() *after* mutex drop (deadlock avoidance).
- module_put(timer->module).

REQ-9: snd_timer_start1(timeri, start, ticks):
- spin_lock_irqsave(&timer->lock).
- EINVAL if IFLG_DEAD.
- ENODEV if card shutting down.
- EBUSY if IFLG_RUNNING | IFLG_START.
- /* Floor check: refuse < 100 µs requested period */
- If start ∧ !SNDRV_TIMER_HW_SLAVE ∧ (resolution × ticks) < 100 000: EINVAL.
- If start: timeri->ticks = timeri->cticks = ticks.
- Else if !cticks: cticks = 1.
- pticks = 0.
- list_move_tail active_list ⊳ timer->active_list_head.
- If timer->running:
  - If HW_SLAVE: jump __start_now.
  - Else: set FLG_RESCHED + IFLG_START → delayed start; return 1.
- Else:
  - If start: timer->sticks = ticks.
  - hw.start(timer).
  - __start_now: timer->running++; set IFLG_RUNNING; return 0.
- notify1(SNDRV_TIMER_EVENT_START / _CONTINUE).

REQ-10: snd_timer_start_slave():
- Hold slave_active_lock.
- EINVAL if IFLG_DEAD; EBUSY if IFLG_RUNNING.
- Set IFLG_RUNNING.
- If master & timer exist: hold master->timer->lock; list_add_tail active_list ⊳ master->slave_active_head; notify1.
- Return 1 (delayed-start).

REQ-11: snd_timer_stop1(timeri, stop):
- Hold timer->lock.
- list_del_init ack_list + active_list.
- EBUSY if neither IFLG_RUNNING nor IFLG_START set.
- If card shutdown: return 0 silently.
- If stop: cticks = ticks; pticks = 0.
- If IFLG_RUNNING ∧ --timer->running == 0:
  - hw.stop(timer).
  - If FLG_RESCHED set: clear, snd_timer_reschedule(timer, 0), if FLG_CHANGE: hw.start(timer).
- Clear IFLG_RUNNING | IFLG_START; set/clear IFLG_PAUSED accordingly.
- notify1(SNDRV_TIMER_EVENT_STOP / _PAUSE).

REQ-12: snd_timer_continue(timeri):
- EINVAL unless IFLG_PAUSED.
- Dispatch to slave/master start1(false, 0).

REQ-13: snd_timer_pause(timeri):
- Dispatch to stop1(false) / stop_slave(false) — preserves cticks.

REQ-14: snd_timer_interrupt(timer, ticks_left):
- Called from backend ISR / hrtimer / mod_timer callback.
- If card shutdown: snd_timer_clear_callbacks(ack_list_head); return.
- Hold timer->lock.
- resolution = c_resolution() ?: hw.resolution.
- For each instance in active_list_head (safe-iter, since callbacks may relink to ack list):
  - Skip if IFLG_DEAD.
  - Skip if !IFLG_RUNNING.
  - pticks += ticks_left; resolution snapshot.
  - cticks -= ticks_left (saturating to 0).
  - If cticks: continue (not expired).
  - If IFLG_AUTO: cticks = ticks (reload).
  - Else: clear IFLG_RUNNING; --timer->running; list_del_init active_list.
  - Pick ack list: fast (`timer->ack_list_head`) if HW_WORK ∨ IFLG_FAST else slow (`sack_list_head`).
  - list_add_tail ti to ack list.
  - For each slave in ti->slave_active_head: propagate pticks + resolution; list_add_tail to same ack list.
- If FLG_RESCHED: snd_timer_reschedule(timer, sticks).
- If timer->running:
  - If HW_STOP: hw.stop(timer); set FLG_CHANGE.
  - If !HW_AUTO ∨ FLG_CHANGE: clear FLG_CHANGE; hw.start(timer).
- Else: hw.stop(timer).
- snd_timer_process_callbacks(timer, &ack_list_head) — fast callbacks under spinlock (release/reacquire across each).
- If !sack_list_head empty: queue_work(system_highpri_wq, &task_work).

REQ-15: snd_timer_process_callbacks(timer, head):
- While !list_empty(head):
  - ti = list_first_entry(head, ack_list).
  - list_del_init &ti->ack_list.
  - If !IFLG_DEAD:
    - Snapshot pticks → ticks (clear pticks), resolution.
    - Set IFLG_CALLBACK.
    - spin_unlock(&timer->lock); ti->callback(ti, resolution, ticks); spin_lock(&timer->lock).
    - Clear IFLG_CALLBACK.

REQ-16: snd_timer_reschedule(timer, ticks_left):
- Walk active_list_head:
  - IFLG_START → IFLG_RUNNING (delayed-start activation); timer->running++.
  - For IFLG_RUNNING: ticks = min(ticks, ti->cticks).
- If no runnable: clear FLG_RESCHED; return.
- Clamp ticks = min(ticks, hw.ticks).
- If ticks ≠ ticks_left: set FLG_CHANGE.
- timer->sticks = ticks.

REQ-17: System-timer backend (jiffies-driven):
- struct snd_timer_system_private { timer_list tlist; snd_timer*; last_expires; last_jiffies; correction; }.
- snd_timer_s_start: njiff = jiffies; apply `correction`; mod_timer(&tlist, njiff).
- snd_timer_s_function (timer_list cb): correction += (jiff - last_expires); snd_timer_interrupt(timer, jiff - last_jiffies).
- snd_timer_s_stop: timer_delete(&tlist); compute residual sticks; correction = 0.
- snd_timer_s_close: timer_delete_sync(&tlist).
- Hardware vtable: SNDRV_TIMER_HW_FIRST | _WORK; resolution = NSEC_PER_SEC / HZ; ticks = 10 000 000L.
- Registered at module init as `system` (SNDRV_TIMER_GLOBAL_SYSTEM) under `snd_timer_global_register`.

REQ-18: hrtimer backend (`sound/core/hrtimer.c`, CONFIG_SND_HRTIMER):
- Per-backend wraps `struct hrtimer` (CLOCK_MONOTONIC) feeding snd_timer_interrupt.
- Registered as SNDRV_TIMER_GLOBAL_HRTIMER.
- Resolution ≈ 1 ns (hardware-clock dependent).

REQ-19: /dev/snd/timer char-dev (`snd_timer_f_ops`):
- minor = SNDRV_MINOR_TIMER; MODULE_ALIAS_CHARDEV(CONFIG_SND_MAJOR, ...).
- open: alloc snd_timer_user (queue_size = 128 default; ioctl_lock; qlock; qchange_sleep wq).
- ioctl: NEXT_DEVICE / GINFO / GPARAMS / GSTATUS / SELECT / INFO / PARAMS / STATUS{32,64} / START / STOP / CONTINUE / PAUSE / TREAD{,64} / CREATE / TRIGGER (utimer).
- read: blocks on qchange_sleep until events; emits `snd_timer_read` (TREAD_FORMAT_NONE) or `snd_timer_tread32` / `_tread64`.
- poll: EPOLLIN | EPOLLRDNORM if qused > 0; EPOLLERR if disconnected.
- release: drains queue; closes underlying instance.
- fasync via `snd_fasync_helper`.

REQ-20: snd_timer_user callbacks (per-fd):
- snd_timer_user_interrupt: per-tick — coalesces same-resolution into last queued entry; else append `snd_timer_read{resolution, ticks}`; on overflow increments `tu->overrun`; wake fasync SIGIO/POLL_IN + qchange_sleep.
- snd_timer_user_ccallback: control-event (START/STOP/CONTINUE/PAUSE + slave M-variants) — append `snd_timer_tread64{event, tstamp, val}`.
- snd_timer_user_tinterrupt: enhanced TREAD with per-tick tstamp.
- snd_timer_user_disconnect: set tu->disconnected; wake pollers (return EPOLLERR / -ENODEV).

REQ-21: snd_timer_notify(timer, event, tstamp):
- Only valid for HW_SLAVE backends (timer->hw.flags & SNDRV_TIMER_HW_SLAVE).
- event ∈ [MSTART, MRESUME].
- Holds timer->lock; walks active_list_head + slave_active_head; invokes ti->ccallback / ts->ccallback.

REQ-22: snd_timer_notify1(ti, event):
- event ∈ [START, PAUSE].
- ktime_get_ts64 (monotonic if timer_tstamp_monotonic) or ktime_get_real_ts64.
- Resolution snapshot for START/CONTINUE.
- ti->ccallback fired; for masters, slave_active_head iterated with event+10 (M-variants).

REQ-23: Userspace-driven timer (CONFIG_SND_UTIMER, `struct snd_utimer`):
- Up to SNDRV_UTIMERS_MAX_COUNT = 128.
- snd_utimer_create(): ida-alloc id; snd_timer_global_new("snd-utimer%i"); install snd_utimer_fops on anon_inode.
- snd_utimer_trigger(): manual ioctl-driven snd_timer_interrupt call.
- snd_utimer_open / _start / _stop: stub hw ops (driven from userspace).

REQ-24: Slave matching:
- check_matching_master_slave(master, slave): compares (slave_class, slave_id); enforces num_instances ≤ max_instances; list_move slave under master->slave_list_head; binds slave->master / slave->timer; if IFLG_RUNNING: insert into master->slave_active_head.
- snd_timer_check_slave / _check_master: pair candidates during open/close.

REQ-25: Per-card-disconnect:
- snd_timer_dev_disconnect: list_del device_list; mark all open instances IFLG_DEAD; invoke each ti->disconnect; wake user pollers (snd_timer_user_disconnect).

REQ-26: Per-timer ID space (struct snd_timer_id):
- dev_class ∈ {NONE, SLAVE, GLOBAL, CARD, PCM}.
- dev_sclass ∈ {NONE, APPLICATION, SEQUENCER, OSS_SEQUENCER}.
- card / device / subdevice integers.
- get_next_device() iterates UAPI NEXT_DEVICE ioctl.

REQ-27: Resolution invariants:
- snd_timer_resolution(timeri): holds timer->lock; returns hw.c_resolution() ?: hw.resolution.
- 0 returned for IFLG_SLAVE-only (slave resolution requested via master).

REQ-28: timer_tstamp_monotonic module param (default 1):
- Selects ktime_get_ts64 (CLOCK_MONOTONIC) vs ktime_get_real_ts64 (CLOCK_REALTIME) for tstamp emissions.

REQ-29: timer_limit module param (default 4 when CONFIG_SND_HRTIMER else 1):
- Bounds global-timer device numbers for `snd-timer-%i` module autoload.

REQ-30: /proc/asound/timers (CONFIG_SND_PROC_FS):
- snd_timer_proc_read walks snd_timer_list; emits per-backend G/C/P prefix + name + resolution + ticks + slave flag + open-list of (owner, running/stopped).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `instance_lists_head_consistent` | INVARIANT | per-instance: open / active / master / ack / slave-list members never aliased to wrong backend. |
| `cticks_monotone_until_expire` | INVARIANT | per-interrupt: cticks only decreases until 0; reload only on AUTO at 0. |
| `iflg_running_implies_active_list_member` | INVARIANT | per-instance: IFLG_RUNNING ⟺ on active_list. |
| `iflg_callback_blocks_close` | INVARIANT | per-close: spins while IFLG_CALLBACK held. |
| `period_floor_100us_enforced` | INVARIANT | per-start: refuses resolution × ticks < 100 000. |
| `slave_master_class_matches` | INVARIANT | per-link: slave.slave_class == master.slave_class ∧ slave.slave_id == master.slave_id. |
| `module_refcount_balanced` | INVARIANT | per-open / close: try_module_get / module_put paired. |
| `card_dev_refcount_balanced` | INVARIANT | per-open / close: get_device / put_device paired. |
| `user_queue_indices_bounded` | INVARIANT | per-tu: qhead, qtail ∈ [0, queue_size). |
| `user_queue_used_accurate` | INVARIANT | per-tu: 0 ≤ qused ≤ queue_size. |

### Layer 2: TLA+

`sound/core/timer.tla`:
- Per-state-machine: NONE → OPEN → RUNNING → STOPPED → CLOSED with PAUSED side-branch.
- Properties:
  - `safety_no_callback_after_close` — per-close: hw.close never fires while IFLG_CALLBACK.
  - `safety_no_double_start` — per-start: EBUSY if already RUNNING / START.
  - `safety_pause_continue_pairing` — per-pause: only IFLG_PAUSED transitions to RUNNING via continue.
  - `safety_slave_only_runs_under_master` — per-slave: IFLG_RUNNING ⟹ master exists ∧ master.timer present.
  - `liveness_interrupt_drains_active` — per-tick: all expired instances eventually on ack list.
  - `liveness_sack_eventually_runs` — per-sack: queued work eventually runs `snd_timer_work`.
  - `safety_register_mutex_serializes_open_close` — per-list: open / close / register / disconnect mutually exclusive.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SndTimer::new` post: hw vtable + lists initialized; sticks = 1; max_instances = 1000 | `SndTimer::new` |
| `SndTimer::open` post: instance attached or err with refcounts restored | `SndTimer::open` |
| `SndTimer::close` post: instance removed; IFLG_CALLBACK cleared; module_put done | `SndTimer::close` |
| `SndTimer::start1` post: cticks = ticks (fresh) ∨ preserved (continue) | `SndTimer::start1` |
| `SndTimer::interrupt` post: expired-non-AUTO leave RUNNING; pticks ≥ ticks_left | `SndTimer::interrupt` |
| `SndTimer::reschedule` post: sticks = min(min(active cticks), hw.ticks) | `SndTimer::reschedule` |
| `SndTimer::process_callbacks` post: head empty; IFLG_CALLBACK cleared per-ti | `SndTimer::process_callbacks` |
| `SndTimerUser::interrupt` post: qused ≤ queue_size; overrun strictly increasing if full | `SndTimerUser::interrupt` |
| `SndTimerUser::read` post: returns N×unit ∨ err; qhead advances by N | `SndTimerUser::read` |

### Layer 4: Verus/Creusot functional

`Per-snd_timer_new → hw.open → snd_timer_open(slave) | snd_timer_open(master) → snd_timer_start → backend tick → snd_timer_interrupt → callback dispatch (fast / sack) → snd_timer_stop → snd_timer_close` semantic equivalence: per-`Documentation/sound/designs/timestamping.rst` and ALSA UAPI semantics in `include/uapi/sound/asound.h`. System backend tick = jiffies-delta(`last_jiffies`); hrtimer backend tick = hrtimer-period; PCM backend tick = pcm.runtime hw-pointer delta.

### hardening

(Inherits row-1 features from `sound/00-overview.md` § Hardening.)

ALSA-timer reinforcement:

- **Per-MAX_SLAVE_INSTANCES = 1000 cap** — defense against per-slave-list exhaustion.
- **Per-timer max_instances = 1000 cap** — defense against per-master open-list exhaustion.
- **Per-100 µs period floor** — defense against per-interrupt-storm via tiny tick.
- **Per-register_mutex serializes register / open / close / disconnect** — defense against per-list TOCTOU.
- **Per-IFLG_CALLBACK barrier on close** — defense against per-callback-after-free.
- **Per-try_module_get/put paired** — defense against per-backend-module unload mid-use.
- **Per-card_dev get/put deferred outside mutex** — defense against per-disconnect deadlock.
- **Per-snd_timer_user disconnect → EPOLLERR / -ENODEV** — defense against per-stale-fd hang.
- **Per-coalescing same-resolution ring entries** — defense against per-queue-overrun from spammy callbacks.
- **Per-system_highpri_wq for slow callbacks** — defense against per-IRQ-context cap on user callbacks.
- **Per-card->shutdown bailout in snd_timer_interrupt** — defense against per-tick after driver gone.
- **Per-overrun counter (not silent drop)** — defense against per-undetected loss.
- **Per-timer_tstamp_monotonic default** — defense against per-realtime-clock-jump tstamp confusion.

