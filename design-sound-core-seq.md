---
title: "Tier-3: sound/core/seq/ — ALSA sequencer (MIDI event routing engine)"
tags: ["tier-3", "sound", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The **ALSA sequencer** is the kernel's MIDI event routing fabric: a userspace or kernel client opens `/dev/snd/seq` (minor `SNDRV_MINOR_SEQUENCER`), allocates a `struct snd_seq_client` with one or more `struct snd_seq_client_port`s, then `subscribes` source-port → destination-port (`snd_seq_port_connect`). Events (`struct snd_seq_event` — 28-byte fixed header + variable-length data) flow through `snd_seq_dispatch_event`, which routes to either USER_CLIENT FIFOs (`struct snd_seq_fifo`) or KERNEL_CLIENT `event_input` callbacks. Time-sensitive events transit through `struct snd_seq_queue` — a pair of priority queues (`tickq` for MIDI ticks, `timeq` for real-time ns) ordered by `snd_seq_prioq_cell_in`, drained on each `snd_seq_check_queue` driven by `struct snd_seq_timer` (wrapper over `snd_timer_instance` from `sound/core/timer.c`). Per-`struct snd_seq_pool` holds the per-client free-list of `struct snd_seq_event_cell` containers (variable-length-event capable). Per-system client (id 0, port `SNDRV_SEQ_PORT_SYSTEM_ANNOUNCE`) broadcasts client-start / client-exit / port-start / port-exit / subscribe / unsubscribe events. OSS sequencer (`sound/core/seq/oss/`) and `snd-seq-midi` (rawmidi bridge) sit as kernel clients. Critical for: ALSA MIDI sequencer apps (rosegarden, qjackctl), JACK MIDI bridges, USB-MIDI / UMP routing, OSS-emul `/dev/sequencer`.

This Tier-3 covers the `sound/core/seq/` subtree centered on `seq.c` initialization plus `seq_clientmgr.c` (the dispatcher) and the supporting `seq_*.c` modules.

### Acceptance Criteria

- [ ] AC-1: alsa_seq_init brings up `/dev/snd/seq` with `snd_seq_f_ops`; failure unrolls in reverse.
- [ ] AC-2: snd_seq_open allocates dynamic client (≥ 128), assigns FIFO if INPUT mode, sends CLIENT_START announce.
- [ ] AC-3: snd_seq_release frees ports + queues + fifo + pool + sends CLIENT_EXIT announce.
- [ ] AC-4: snd_seq_create_port assigns port 0..253; appears via SNDRV_SEQ_IOCTL_QUERY_NEXT_PORT.
- [ ] AC-5: snd_seq_port_connect with mismatching SUBS_READ / SUBS_WRITE caps returns -EPERM.
- [ ] AC-6: PORT_SUBSCRIBED announce delivered to system ANNOUNCE port after successful connect.
- [ ] AC-7: snd_seq_dispatch_event with hop > 8 returns -EINVAL (loop guard).
- [ ] AC-8: deliver_to_subscribers fans out to every subscriber in c_src.list_head with hop+1.
- [ ] AC-9: USER → USER write through SUBSCRIBERS dest delivers to dest FIFO; read returns the event.
- [ ] AC-10: USER → KERNEL_CLIENT delivery invokes port->event_input synchronously.
- [ ] AC-11: snd_seq_enqueue_event with TIME_STAMP_TICK inserts into q->tickq sorted by tick.
- [ ] AC-12: snd_seq_check_queue drains all ready cells (cur_time ≥ event.time) then re-loops on check_again.
- [ ] AC-13: SNDRV_SEQ_EVENT_TEMPO event reprograms queue tempo / tick resolution.
- [ ] AC-14: snd_seq_event_dup blocks on output_sleep when pool full ∧ !nonblock.
- [ ] AC-15: snd_seq_fifo overflow sets atomic overflow flag; snd_seq_read returns -ENOSPC then clears.
- [ ] AC-16: bounce_error_event delivers KERNEL_ERROR quote only when FILTER_BOUNCE ∧ accept_input.
- [ ] AC-17: snd_seq_queue_alloc returns -ENOSPC after SNDRV_SEQ_MAX_QUEUES=32 queues created.
- [ ] AC-18: SNDRV_SEQ_MAX_CLIENTS=192 cap: 193rd open returns -ENOMEM.

### Architecture

```
struct SndSeqClient {
  type: SeqClientType,                  // NO_CLIENT | USER | KERNEL
  accept_input: bool,
  accept_output: bool,
  midi_version: u32,                    // LEGACY | UMP_1_0 | UMP_2_0
  user_pversion: u32,
  name: [u8; 64],
  number: i32,
  filter: u32,
  event_filter: BitMap<256>,
  group_filter: u32,
  use_lock: SndUseLock,
  event_lost: AtomicI32,
  num_ports: i32,
  ports_list_head: ListHead,
  ports_lock: RwLock<()>,
  ports_mutex: Mutex<()>,
  ioctl_mutex: Mutex<()>,
  pool: Option<Box<SndSeqPool>>,
  data: SndSeqClientData,               // enum { User { file, owner, fifo, fifo_pool_size }, Kernel { card } }
  ump_info: Option<Vec<NonNull<()>>>,
  ump_endpoint_port: i32,
}

struct SndSeqClientPort {
  addr: SndSeqAddr,                     // { client: u8, port: u8 }
  owner: Option<*Module>,
  name: [u8; 64],
  list: ListHead,
  use_lock: SndUseLock,
  c_src: SndSeqPortSubsInfo,            // sender edges (this is source)
  c_dest: SndSeqPortSubsInfo,           // receiver edges (this is dest)
  event_input: Option<fn(&SndSeqEvent, i32, NonNull<()>, i32, i32) -> i32>,
  private_data: Option<NonNull<()>>,
  private_free: Option<fn(NonNull<()>)>,
  closing: bool,
  timestamping: bool,
  time_real: bool,
  time_queue: i32,
  capability: u32,                      // SNDRV_SEQ_PORT_CAP_*
  type_: u32,                           // SNDRV_SEQ_PORT_TYPE_*
  midi_channels: i32,
  midi_voices: i32,
  synth_voices: i32,
  direction: u8,
  ump_group: u8,
  is_midi1: bool,
}

struct SndSeqQueue {
  queue: i32,
  name: [u8; 64],
  tickq: Box<SndSeqPrioq>,              // MIDI-tick domain
  timeq: Box<SndSeqPrioq>,              // real-time domain
  timer: Box<SndSeqTimer>,
  owner: i32,
  locked: bool,
  klocked: bool,                        // running flag (after START)
  check_again: bool,
  check_blocked: bool,
  flags: u32,
  info_flags: u32,
  owner_lock: SpinLock<()>,
  check_lock: SpinLock<()>,
  clients_bitmap: BitMap<192>,
  clients: u32,
  timer_mutex: Mutex<()>,
  use_lock: SndUseLock,
}

struct SndSeqPrioq {
  head: Option<*SndSeqEventCell>,
  tail: Option<*SndSeqEventCell>,
  cells: AtomicI32,
  lock: SpinLock<()>,
}

struct SndSeqPool {
  ptr: Box<[SndSeqEventCell]>,
  free: Option<*SndSeqEventCell>,
  total_elements: i32,
  counter: AtomicI32,
  size: i32,
  room: i32,
  closing: bool,
  output_sleep: WaitQueueHead,
  lock: SpinLock<()>,
}

struct SndSeqEvent {                    // UAPI, 28 bytes
  type_: u8,
  flags: u8,
  tag: i8,
  queue: u8,
  time: SndSeqTimestamp,                // union { tick: u32, time: { sec: u32, nsec: u32 } }
  source: SndSeqAddr,
  dest: SndSeqAddr,
  data: SndSeqEventData,                // 12-byte union
}

struct SndSeqEventCell {
  event: SndSeqEvent,
  pool: NonNull<SndSeqPool>,
  next: Option<*SndSeqEventCell>,       // chain for variable-length
}

struct SndSeqTimer {
  running: bool,
  tempo: i32,                           // ns per beat
  ppq: i32,
  skew: u32,
  skew_base: u32,
  preferred_resolution: u32,
  ticks: u64,
  tick: SndSeqTimerTick,                // { fraction, cur_resolution, resolution }
  cur_time: Timespec64,
  last_update: u64,
  type_: i32,
  sclass: i32,
  card: i32,
  device: i32,
  subdevice: i32,
  timeri: Option<*SndTimerInstance>,    // from sound/core/timer.c
}
```

`SndSeqClient::dispatch_event(cell, atomic, hop) -> i32`:
1. If hop >= SNDRV_SEQ_MAX_HOPS (8): return -EINVAL.
2. ev = &cell.event.
3. src = clientptr(ev.source.client).
4. If ev.dest.client == SNDRV_SEQ_ADDRESS_SUBSCRIBERS:
   - sport = snd_seq_port_use_ptr(src, ev.source.port).
   - if !sport: snd_seq_cell_free(cell); return -EINVAL.
   - SndSeqClient::deliver_to_subscribers(src, sport, ev, atomic, hop+1).
   - snd_seq_port_unlock(sport).
5. Else:
   - SndSeqClient::deliver_single_event(src, ev, atomic, hop).
6. snd_seq_cell_free(cell).
7. return 0.

`SndSeqClient::deliver_single_event(src, ev, atomic, hop) -> i32`:
1. dest = get_event_dest_client(ev).
2. If !dest:
   - bounce_error_event(src, ev, -ENOENT, atomic, hop).
   - return -ENOENT.
3. dest_port = snd_seq_port_use_ptr(dest, ev.dest.port).
4. If !dest_port: ... bounce.
5. err = __snd_seq_deliver_single_event(dest, dest_port, ev, atomic, hop).
6. If err < 0: bounce_error_event(src, ev, err, atomic, hop).
7. snd_seq_port_unlock(dest_port); snd_seq_client_unref(dest).

`SndSeqClient::deliver_to_subscribers(src, sport, ev, atomic, hop) -> i32`:
1. Hold sport.c_src.list_lock (read).
2. For sub in sport.c_src.list_head:
   - copy ev → tmp (so per-subscriber dest stamp ok).
   - tmp.dest = sub.info.dest.
   - SndSeqClient::deliver_single_event(src, &tmp, atomic, hop).

`SndSeqQueue::enqueue_event(cell, atomic, hop) -> i32`:
1. ev = &cell.event.
2. q = queueptr(ev.queue) or return -EINVAL.
3. /* Resolve time */
4. If ev.flags & SNDRV_SEQ_TIME_MODE_REL:
   - cur_tick = snd_seq_timer_get_cur_tick(q.timer).
   - cur_time = snd_seq_timer_get_cur_time(q.timer, true).
   - ev.time = absolute(ev.time + cur).
5. If (ev.flags & SNDRV_SEQ_TIME_STAMP_MASK) == SNDRV_SEQ_TIME_STAMP_TICK:
   - SndSeqPrioq::cell_in(q.tickq, cell).
6. Else:
   - SndSeqPrioq::cell_in(q.timeq, cell).
7. snd_seq_check_queue(q, atomic, hop).
8. queuefree(q).

`SndSeqQueue::check(q, atomic, hop)`:
1. If !q.klocked: return.
2. Hold q.check_lock.
3. If q.check_blocked: q.check_again = true; return.
4. q.check_blocked = true; drop check_lock.
5. Loop:
   - cur_tick = snd_seq_timer_get_cur_tick(q.timer).
   - cur_time = snd_seq_timer_get_cur_time(q.timer, true).
   - Drain tickq: while cell = prioq_cell_out(tickq, &cur_tick): snd_seq_dispatch_event(cell, atomic, hop).
   - Drain timeq: while cell = prioq_cell_out(timeq, &cur_time): dispatch.
6. Hold check_lock; if q.check_again: clear, loop again.
7. q.check_blocked = false.

`SndSeqPrioq::cell_in(f, cell)`:
1. Hold f.lock.
2. Find insertion point — walk from tail toward head until cell.time ≤ candidate.time (insertion-sort).
3. Splice cell at insertion point.
4. f.cells += 1.

`SndSeqTimer::interrupt(timeri, resolution, ticks)`:
1. q = timeri.callback_data as &SndSeqQueue.
2. tmr = q.timer.
3. Hold tmr.lock.
4. /* Advance real-time accumulator */
5. tmr.cur_time.nsec += ticks * resolution.
6. Normalize sec/nsec.
7. /* Advance tick accumulator */
8. delta_frac = ticks * tmr.tick.resolution.
9. tmr.tick.fraction += delta_frac.
10. tmr.ticks += tmr.tick.fraction >> 16; tmr.tick.fraction &= 0xffff.
11. snd_seq_check_queue(q, atomic=1, hop=0).

### Out of Scope

- sound/core/seq/oss/ OSS sequencer compat (covered separately if expanded)
- sound/core/seq/seq_ump_convert.c MIDI 2.0 / UMP conversion (covered in `seq-ump.md` if expanded)
- sound/core/seq/seq_midi_emul.c MIDI-emul (covered separately if expanded)
- sound/core/timer.c ALSA timer abstract layer (covered in `timer.md` Tier-3)
- sound/core/rawmidi.c rawmidi driver (covered in `rawmidi.md` Tier-3)
- include/uapi/sound/asequencer.h UAPI layouts (covered in `sound/uapi/asequencer.md` if expanded)
- seq_compat.c 32-bit compat shims (covered with SYSCALL_COMPAT track)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct snd_seq_client` | per-client (USER or KERNEL) | `SndSeqClient` |
| `struct snd_seq_user_client` | per-userspace state (file, fifo, owner pid) | `SndSeqUserClient` |
| `struct snd_seq_kernel_client` | per-kernel state (card backref) | `SndSeqKernelClient` |
| `struct snd_seq_client_port` | per-port | `SndSeqClientPort` |
| `struct snd_seq_port_subs_info` | per-port subscriber group | `SndSeqPortSubsInfo` |
| `struct snd_seq_subscribers` | per-edge subscription record | `SndSeqSubscribers` |
| `struct snd_seq_event` | per-event (28 bytes UAPI) | `SndSeqEvent` |
| `struct snd_seq_event_cell` | per-event ring-cell with next-ptr | `SndSeqEventCell` |
| `struct snd_seq_pool` | per-client cell pool | `SndSeqPool` |
| `struct snd_seq_fifo` | per-USER input FIFO | `SndSeqFifo` |
| `struct snd_seq_queue` | per-queue (tickq + timeq + timer) | `SndSeqQueue` |
| `struct snd_seq_prioq` | per-priority queue (time-ordered) | `SndSeqPrioq` |
| `struct snd_seq_timer` | per-queue timer wrapper | `SndSeqTimer` |
| `alsa_seq_init()` / `alsa_seq_exit()` | per-module load/unload | `SndSeq::init` / `exit` |
| `client_init_data()` | per-clienttab zap | `SndSeq::client_init_data` |
| `snd_sequencer_device_init()` / `_done()` | per-/dev/snd/seq registration | `SndSeq::device_init` / `done` |
| `snd_seq_info_init()` / `_done()` | per-/proc/asound/seq | `SndSeq::info_init` / `done` |
| `snd_seq_system_client_init()` / `_done()` | per-system-client (id 0) | `SndSeq::system_client_init` / `done` |
| `snd_seq_autoload_init()` / `_exit()` | per-CONFIG_SND_SEQ_DUMMY_MODULE autoload | `SndSeq::autoload_init` / `exit` |
| `snd_seq_queues_delete()` | per-exit drain queues | `SndSeq::queues_delete` |
| `snd_seq_open()` / `_release()` / `_read()` / `_write()` / `_poll()` / `_ioctl()` | per-/dev/snd/seq fops | `SndSeqClient::file_ops` |
| `seq_create_client1()` / `seq_free_client1()` / `seq_free_client()` | per-client lifecycle | `SndSeqClient::new` / `drop` |
| `snd_seq_create_kernel_client()` / `_delete_kernel_client()` | per-kernel-client | `SndSeqClient::new_kernel` / `delete_kernel` |
| `snd_seq_dispatch_event()` | per-event routing core | `SndSeqClient::dispatch_event` |
| `__snd_seq_deliver_single_event()` | per-dest-delivery | `SndSeqClient::deliver_single_event_inner` |
| `snd_seq_deliver_event()` | per-direct or subscribe | `SndSeqClient::deliver_event` |
| `deliver_to_subscribers()` | per-fan-out | `SndSeqClient::deliver_to_subscribers` |
| `snd_seq_client_enqueue_event()` | per-write path | `SndSeqClient::enqueue_event` |
| `bounce_error_event()` | per-error bounce | `SndSeqClient::bounce_error` |
| `snd_seq_create_port()` / `_delete_port()` / `_delete_all_ports()` | per-port lifecycle | `SndSeqClientPort::*` |
| `snd_seq_port_connect()` / `_disconnect()` | per-subscribe / unsubscribe | `SndSeqClientPort::connect` / `disconnect` |
| `check_and_subscribe_port()` / `__delete_and_unsubscribe_port()` | per-edge primitive | `SndSeqClientPort::edge_*` |
| `snd_seq_queue_alloc()` / `_delete()` | per-queue | `SndSeqQueue::alloc` / `delete` |
| `snd_seq_queue_use()` / `_is_used()` | per-client refcount | `SndSeqQueue::use` / `is_used` |
| `snd_seq_queue_timer_open()` / `_close()` / `_set_tempo()` | per-queue timer ctl | `SndSeqQueue::timer_*` |
| `snd_seq_enqueue_event()` | per-queue-enqueue | `SndSeqQueue::enqueue_event` |
| `snd_seq_check_queue()` | per-tick drain | `SndSeqQueue::check` |
| `snd_seq_control_queue()` | per-START/STOP/CONT/SETPOS event | `SndSeqQueue::control` |
| `snd_seq_prioq_cell_in()` / `_out()` | per-time-ordered insert / pop | `SndSeqPrioq::cell_in` / `cell_out` |
| `snd_seq_prioq_leave()` / `_remove_events()` | per-client cleanup | `SndSeqPrioq::leave` / `remove_events` |
| `snd_seq_fifo_new()` / `_delete()` / `_event_in()` / `_cell_out()` / `_resize()` | per-USER FIFO | `SndSeqFifo::*` |
| `snd_seq_pool_new()` / `_init()` / `_done()` / `_delete()` | per-pool | `SndSeqPool::*` |
| `snd_seq_event_dup()` | per-event-clone | `SndSeqPool::event_dup` |
| `snd_seq_cell_free()` | per-cell-return | `SndSeqPool::cell_free` |
| `snd_seq_timer_new()` / `_delete()` / `_open()` / `_close()` / `_start()` / `_stop()` / `_continue()` / `_interrupt()` | per-queue timer | `SndSeqTimer::*` |
| `snd_seq_timer_set_tempo()` / `_set_tempo_ppq()` / `_set_position_tick()` / `_set_position_time()` / `_set_skew()` | per-queue tempo / position | `SndSeqTimer::set_*` |
| `snd_seq_kernel_client_enqueue()` / `_dispatch()` / `_ctl()` / `_ioctl()` | per-kernel-side API | `SndSeqClient::kernel_*` |

### compatibility contract

REQ-1: alsa_seq_init() — ordered bring-up:
- /* Step 1 */ client_init_data() — zero clienttab[] + clienttablock[].
- /* Step 2 */ snd_sequencer_device_init() — `snd_register_device(SNDRV_DEVICE_TYPE_SEQUENCER, NULL, 0, &snd_seq_f_ops, NULL, dev)` exposing `/dev/snd/seq`.
- /* Step 3 */ snd_seq_info_init() — /proc/asound/seq/{clients, queues, timer}.
- /* Step 4 */ snd_seq_system_client_init() — register system client id 0 with announce port 0 (`SNDRV_SEQ_PORT_SYSTEM_ANNOUNCE`) and timer port 1 (`SNDRV_SEQ_PORT_SYSTEM_TIMER`).
- /* Step 5 */ snd_seq_autoload_init() — kick off autoload of `snd-seq-dummy` when `CONFIG_SND_SEQ_DUMMY_MODULE`.
- Failure unrolls: info_done → device_done → return err.

REQ-2: alsa_seq_exit() — reverse order:
- snd_seq_system_client_done() → snd_seq_info_done() → snd_seq_queues_delete() (drain all queues) → snd_sequencer_device_done() → snd_seq_autoload_exit().

REQ-3: Module params (seq.c):
- `seq_client_load[15]`: list of global clients (0..14) to load at init (default first slot = SNDRV_SEQ_CLIENT_DUMMY when dummy-module configured).
- `seq_default_timer_class` (default GLOBAL), `_sclass` (default SCLASS_NONE), `_card` (-1), `_device` (HRTIMER if CONFIG_SND_SEQ_HRTIMER_DEFAULT else SYSTEM), `_subdevice` (0), `_resolution` (Hz, 0 = backend default).

REQ-4: Client number ranges (per `seq_clientmgr.c` comment):
- 0..15 — global / kernel clients (system = 0, OSS = 1, midi-emul = 14 etc.).
- 16..127 — statically allocated for cards 0..27 (`SNDRV_SEQ_CLIENTS_PER_CARD = 4`).
- 128..191 — dynamic clients (kernel drivers for cards 28..31 + user apps).
- Total `SNDRV_SEQ_MAX_CLIENTS = 192`.

REQ-5: struct snd_seq_client:
- type: NO_CLIENT | USER_CLIENT | KERNEL_CLIENT.
- accept_input / accept_output: 1-bit flags from file mode.
- midi_version: SNDRV_SEQ_CLIENT_LEGACY_MIDI / UMP_MIDI_1_0 / UMP_MIDI_2_0.
- user_pversion: per-userspace API version.
- name[64], number, filter, event_filter[256-bit bitmap], group_filter.
- use_lock: snd_use_lock_t (atomic refcount).
- event_lost: counter.
- num_ports, ports_list_head, ports_lock (rwlock), ports_mutex.
- ioctl_mutex.
- pool: snd_seq_pool* (per-client output pool).
- data.user.{file, owner pid, fifo, fifo_pool_size} or data.kernel.{card}.
- ump_info / ump_endpoint_port (UMP).

REQ-6: struct snd_seq_client_port:
- addr: snd_seq_addr {client, port}.
- owner: module.
- name[64], list (link in client.ports_list_head), use_lock.
- c_src / c_dest: snd_seq_port_subs_info subscriber groups (sender / receiver edges).
- event_input(ev, direct, priv, atomic, hop): KERNEL_CLIENT callback.
- private_data, private_free.
- closing / timestamping / time_real bits.
- time_queue: associated queue id.
- capability: SNDRV_SEQ_PORT_CAP_{READ,WRITE,SYNC_READ,SYNC_WRITE,DUPLEX,SUBS_READ,SUBS_WRITE,NO_EXPORT}.
- type: SNDRV_SEQ_PORT_TYPE_{SPECIFIC,MIDI_GENERIC,MIDI_GM,MIDI_GS,MIDI_XG,MIDI_MT32,...,APPLICATION}.
- midi_channels / midi_voices / synth_voices.
- direction / ump_group / is_midi1 (UMP).

REQ-7: struct snd_seq_event (UAPI, 28 bytes):
- type: u8 — SNDRV_SEQ_EVENT_NOTEON / NOTEOFF / KEYPRESS / CONTROLLER / PGMCHANGE / CHANPRESS / PITCHBEND / SYSEX / ECHO / OSS / CLIENT_START / CLIENT_EXIT / PORT_START / PORT_EXIT / PORT_SUBSCRIBED / PORT_UNSUBSCRIBED / QUEUE_SKEW / TEMPO / TICK / CLOCK / START / CONTINUE / STOP / SETPOS_TICK / SETPOS_TIME / BOUNCE / KERNEL_ERROR / NONE.
- flags: u8 — TIME_STAMP_{TICK,REAL}, TIME_MODE_{ABS,REL}, PRIORITY_{NORMAL,HIGH}, LENGTH_{FIXED,VARIABLE,VARUSR}.
- tag: u8.
- queue: u8 — SNDRV_SEQ_QUEUE_DIRECT (253) for direct delivery; else queue id.
- time: union {tick u32; time {sec u32; nsec u32}}.
- source: snd_seq_addr.
- dest: snd_seq_addr.
- data: union (note / control / queue / addr / connect / result / ext / raw32 / raw8) — 12 bytes.

REQ-8: snd_seq_open(inode, file):
- stream_open(inode, file).
- Hold register_mutex.
- seq_create_client1(-1, SNDRV_SEQ_DEFAULT_EVENTS=500) — allocate dynamic client number.
- mode = file flags → SNDRV_SEQ_LFLG_INPUT/OUTPUT.
- If input: allocate snd_seq_fifo with SNDRV_SEQ_DEFAULT_CLIENT_EVENTS=200.
- usage_alloc(&client_usage, 1).
- client->type = USER_CLIENT; client->name = "Client-<n>".
- client->data.user.owner = get_pid(task_pid(current)).
- snd_seq_system_client_ev_client_start(c) — broadcast.

REQ-9: snd_seq_release(inode, file):
- seq_free_client(client) → seq_free_client1 → snd_seq_delete_all_ports + snd_seq_queue_client_leave + snd_use_lock_sync + pool_delete.
- snd_seq_fifo_delete(&fifo).
- free_ump_info (UMP).
- put_pid(owner); kfree(client).
- snd_seq_system_client_ev_client_exit(c).

REQ-10: snd_seq_write(file, buf, count):
- Per-event copy_from_user (sizeof(snd_seq_event) or snd_seq_ump_event when client->midi_version).
- check_event_type_and_length: validate type ∈ valid range, length flags consistent with var data.
- snd_seq_client_enqueue_event(client, ev, file, blocking, atomic=false, hop=0):
  - Stamp ev->source.client = client->number.
  - If dest == SUBSCRIBERS or fixed: snd_seq_deliver_event.
  - Else if direct: bypass queue.
  - Else: snd_seq_event_dup(pool, ev, &cell, nonblock, file, &ioctl_mutex) → snd_seq_enqueue_event(cell, atomic, hop).

REQ-11: snd_seq_dispatch_event(cell, atomic, hop):
- hop guard: EINVAL if hop > SNDRV_SEQ_MAX_HOPS (8).
- Resolve source client via clientptr(ev.source.client).
- Resolve dest via get_event_dest_client.
- /* Single vs broadcast */
- If dest.client == SUBSCRIBERS or PORT_SUBSCRIBERS: deliver_to_subscribers(client, ev, atomic, hop).
- Else: snd_seq_deliver_single_event(client, ev, atomic, hop).
- snd_seq_cell_free(cell) after delivery.

REQ-12: __snd_seq_deliver_single_event(dest, dest_port, ev, atomic, hop):
- Switch dest->type:
  - USER_CLIENT: snd_seq_fifo_event_in(dest->data.user.fifo, ev).
  - KERNEL_CLIENT: dest_port->event_input(ev, direct, dest_port->private_data, atomic, hop).
  - NO_CLIENT: drop.

REQ-13: deliver_to_subscribers(client, ev, atomic, hop):
- Look up source port from ev.source.port; iterate port->c_src.list_head:
  - For each subscriber, snd_seq_deliver_single_event_to(dest_addr, ev, atomic, hop+1).
- On error: bounce_error_event back to source (when source.filter & SNDRV_SEQ_FILTER_BOUNCE).

REQ-14: bounce_error_event(client, event, err, atomic, hop):
- Skip if !(filter & SNDRV_SEQ_FILTER_BOUNCE) or !client->accept_input.
- Construct SNDRV_SEQ_EVENT_KERNEL_ERROR cell quoting origin + value=-err.
- snd_seq_deliver_single_event(NULL, &bounce_ev, atomic, hop+1).
- Increment client->event_lost on failure.

REQ-15: snd_seq_create_port(client, port_index, &port_ret):
- Allocate snd_seq_client_port; init c_src + c_dest port_subs_info with rwlock + rwsem.
- Assign port number (port_index or next free 0..SNDRV_SEQ_MAX_PORTS-1=253).
- list_add port to client->ports_list_head under ports_lock + ports_mutex.
- client->num_ports += 1.

REQ-16: snd_seq_port_connect(connector, src, sp, dest, dp, info):
- check_subscription_permission: validate capability bits (src.SUBS_READ ∧ dest.SUBS_WRITE; or NO_EXPORT path).
- check_and_subscribe_port (twice — once src side, once dest side, with paired unwind on failure):
  - Hold src->c_src.list_mutex.
  - Allocate snd_seq_subscribers struct (atomic_t ref_count = 2).
  - list_add subscribers->src_list ⊳ src->c_src.list_head.
  - Repeat symmetric for dest->c_dest.
  - src->c_src.open + dest->c_dest.open callbacks fired.
- snd_seq_client_notify_subscription(SNDRV_SEQ_EVENT_PORT_SUBSCRIBED) — broadcast to system port.

REQ-17: snd_seq_port_disconnect(connector, src, sp, dest, dp, info):
- Find matching subscribers via match_subs_info.
- __delete_and_unsubscribe_port for both sides (decrement ref_count; kfree at 0).
- Notify SNDRV_SEQ_EVENT_PORT_UNSUBSCRIBED.

REQ-18: struct snd_seq_queue (per `seq_queue.h`):
- queue: id.
- name[64].
- tickq: snd_seq_prioq* for tick-domain events.
- timeq: snd_seq_prioq* for real-time-domain events.
- timer: snd_seq_timer*.
- owner: client id; locked: bool (timer accessible only by owner).
- klocked: kernel-START lock.
- check_again / check_blocked: re-entrancy on snd_seq_check_queue.
- flags / info_flags.
- owner_lock / check_lock: spinlocks.
- clients_bitmap[SNDRV_SEQ_MAX_CLIENTS]: bitmap of using clients.
- clients: ref count.
- timer_mutex: serialize timer ctl.
- use_lock: snd_use_lock_t.

REQ-19: snd_seq_queue_alloc(client, locked, info_flags):
- queue_new(client, locked) — allocate; alloc tickq + timeq + timer.
- queue_list_add — assign id from SNDRV_SEQ_MAX_QUEUES=32 free slot; EBUSY if full.
- Return queue (use_lock-held caller pattern).

REQ-20: snd_seq_enqueue_event(cell, atomic, hop):
- ev = &cell->event.
- q = queueptr(ev.queue); EINVAL if !q.
- Resolve cur_time / cur_tick via snd_seq_timer_get_cur_time / _get_cur_tick.
- If ev.flags & TIME_MODE_REL: convert relative to absolute.
- If TIME_STAMP_TICK: snd_seq_prioq_cell_in(q->tickq, cell).
- Else (TIME_STAMP_REAL): snd_seq_prioq_cell_in(q->timeq, cell).
- snd_seq_check_queue(q, atomic, hop).

REQ-21: snd_seq_check_queue(q, atomic, hop):
- Bail if !q->klocked (queue not running).
- Hold check_lock; if check_blocked: set check_again; return.
- Pop ready cells (snd_seq_prioq_cell_out comparing against cur_tick / cur_time) — for each: snd_seq_dispatch_event(cell, atomic, hop).
- Re-loop if check_again set during dispatch.

REQ-22: snd_seq_prioq_cell_in(f, cell):
- /* Insertion-sort by timestamp into doubly-linked list */
- Find insertion point comparing (cell->event.time.tick / .time.time vs existing).
- Insert before first cell with later time.
- f->cells += 1.

REQ-23: snd_seq_prioq_cell_out(f, current_time):
- Pop head cell if event_is_ready(head, current_time).
- event_is_ready: head->event.time ≤ *current_time (compare_timestamp).
- Return cell or NULL.

REQ-24: snd_seq_prioq_leave(f, client, timestamp):
- Walk f->head; remove any cell where event.source.client == client (∧ optional timestamp match).
- snd_seq_cell_free(cell) per removal.

REQ-25: struct snd_seq_timer (per `seq_timer.h`):
- running: bool.
- tempo: int (ns per beat).
- ppq: int (ticks per quarter note).
- skew / skew_base: tempo skew.
- preferred_resolution: hint.
- ticks: current tick count.
- tick: current tick fractional (resolution-tracked).
- cur_time: real-time accumulator.
- last_update: jiffies/ktime snapshot.
- type, sclass, card, device, subdevice: backing snd_timer_id.
- timeri: bound snd_timer_instance*.

REQ-26: snd_seq_timer_open(q):
- Set up snd_timer_id from queue tempo/timer params.
- snd_timer_instance_new + snd_timer_open → bind.
- timeri->callback = snd_seq_timer_interrupt.
- timeri->callback_data = q.
- snd_seq_timer_set_tick_resolution() — compute tick resolution from ppq * tempo.
- ENOMEM / EINVAL propagation.

REQ-27: snd_seq_timer_interrupt(timeri, resolution, ticks):
- q = timeri->callback_data.
- Update tmr->cur_time (real ns) += ticks × resolution.
- Update tmr->tick.fraction; carry into tmr->ticks (integer tick increments).
- snd_seq_check_queue(q, atomic=1, hop=0) — drain ready events under IRQ context.

REQ-28: snd_seq_timer_set_tempo / _set_tempo_ppq:
- EINVAL if tempo ≤ 0 ∨ ppq ≤ 0.
- Hold q->timer_mutex.
- Recompute tick resolution.

REQ-29: snd_seq_control_queue(ev, atomic, hop):
- For SNDRV_SEQ_EVENT_START / CONTINUE / STOP / SETPOS_TICK / SETPOS_TIME / TEMPO / QUEUE_SKEW:
  - Validate access via snd_seq_queue_check_access(qid, src.client).
  - Dispatch to snd_seq_timer_start / _continue / _stop / _set_position_tick / _set_position_time / _set_tempo / _set_skew.

REQ-30: struct snd_seq_pool:
- ptr: pre-allocated array of snd_seq_event_cell (size = pool->size).
- free: head of free-list (intrusive next).
- total_elements: pool->size.
- counter: atomic_t — cells in use.
- room: low-watermark for wakeup.
- closing: drain flag.
- output_sleep: wait_queue for blocking writes.
- lock: spinlock.

REQ-31: snd_seq_event_dup(pool, event, &cellp, nonblock, file, &mutexp):
- Allocate cell from pool->free (snd_seq_cell_alloc — blocks on output_sleep if !nonblock; releases mutex across schedule).
- Copy event header.
- If variable length: chained cells (`SNDRV_SEQ_EVENT_LENGTH_VARIABLE`) — allocate continuation cells; copy var-data.
- Set ev.flags chain bit.

REQ-32: snd_seq_cell_free(cell):
- Hold pool->lock.
- If chained: recursively free chained continuation cells.
- Return cell to free-list; atomic_dec(&pool->counter).
- wake_up(&pool->output_sleep) if writer waiting on room.

REQ-33: snd_seq_fifo (per-USER input FIFO):
- pool: dedicated snd_seq_pool.
- head / tail / cells (atomic).
- input_sleep: wait_queue for snd_seq_read.
- overflow: atomic_t — set on full → snd_seq_read returns -ENOSPC.
- snd_seq_fifo_event_in: cell_dup into fifo; on full: atomic_inc(&overflow).
- snd_seq_fifo_cell_out: blocking pop (releases fifo lock across schedule).

REQ-34: snd_seq_system_client (id 0):
- Two well-known ports:
  - SNDRV_SEQ_PORT_SYSTEM_TIMER (port 0).
  - SNDRV_SEQ_PORT_SYSTEM_ANNOUNCE (port 1) — receive-only, broadcasts CLIENT_START / CLIENT_EXIT / PORT_START / PORT_EXIT / PORT_SUBSCRIBED / PORT_UNSUBSCRIBED.
- snd_seq_system_client_ev_client_start/_exit/_port_start/_port_exit: enqueue announce.

REQ-35: snd_seq_kernel_client API (header `<sound/seq_kernel.h>`):
- snd_seq_create_kernel_client(card, client_index, name) → integer client number.
- snd_seq_delete_kernel_client(clientid).
- snd_seq_kernel_client_enqueue(clientid, ev, file, blocking).
- snd_seq_kernel_client_dispatch(clientid, ev, atomic, hop) — direct dispatch bypassing queue.
- snd_seq_kernel_client_ctl(clientid, cmd, arg) — invoke ioctls from kernel context.
- snd_seq_kernel_client_get(id) / _put(client) — refcounted ptr.
- Used by `snd-seq-midi` (rawmidi bridge), `snd-seq-dummy`, OSS sequencer, virmidi, UMP client.

REQ-36: /dev/snd/seq char-dev fops (`snd_seq_f_ops`):
- open: snd_seq_open.
- release: snd_seq_release.
- read: snd_seq_read (drains USER fifo → user buffer with optional var-event expansion).
- write: snd_seq_write (validate + enqueue).
- poll: snd_seq_poll (EPOLLIN if fifo has cells / EPOLLOUT if pool has room).
- ioctl (`snd_seq_ioctl`):
  - PVERSION / CLIENT_ID / SYSTEM_INFO / RUNNING_MODE.
  - GET_CLIENT_INFO / SET_CLIENT_INFO.
  - CREATE_PORT / DELETE_PORT / GET_PORT_INFO / SET_PORT_INFO.
  - SUBSCRIBE_PORT / UNSUBSCRIBE_PORT.
  - CREATE_QUEUE / DELETE_QUEUE / GET_QUEUE_INFO / SET_QUEUE_INFO / GET_NAMED_QUEUE.
  - GET_QUEUE_STATUS / GET_QUEUE_TEMPO / SET_QUEUE_TEMPO / GET_QUEUE_TIMER / SET_QUEUE_TIMER.
  - GET_QUEUE_CLIENT / SET_QUEUE_CLIENT.
  - GET_CLIENT_POOL / SET_CLIENT_POOL.
  - REMOVE_EVENTS / GET_SUBSCRIPTION / QUERY_SUBS / QUERY_NEXT_CLIENT / QUERY_NEXT_PORT.
  - CLIENT_UMP_INFO (UMP).
- compat_ioctl: `seq_compat.c` 32↔64 translations.
- fasync.

REQ-37: snd_seq_oss (/dev/sequencer + /dev/music — `sound/core/seq/oss/`):
- Registers kernel client SNDRV_SEQ_CLIENT_OSS=1.
- Translates OSS sequencer events to ALSA events.

REQ-38: snd_seq_midi (`seq_midi.c`):
- Registered as kernel client per snd-rawmidi card.
- Pumps rawmidi bytes ⇄ snd_seq_events.

REQ-39: snd_seq_virmidi (`seq_virmidi.c`):
- Virtual rawmidi device acting as sequencer client.

REQ-40: snd_seq_dummy (`seq_dummy.c`, CONFIG_SND_SEQ_DUMMY):
- Loopback ports for testing — copies events from input subscribers to output subscribers.

REQ-41: snd_use_lock (`seq_lock.c`):
- atomic_t + wait_queue.
- snd_use_lock_init / _use / _free / _sync.
- _sync blocks until all references released — used at client/port/queue/pool teardown to fence reader paths.

REQ-42: Hop guard (SNDRV_SEQ_MAX_HOPS=8):
- Every dispatch path increments hop.
- snd_seq_dispatch_event returns -EINVAL when hop > MAX — prevents subscription cycles from infinite-loop.

REQ-43: snd_seq_remove_events ioctl:
- Remove events from output pool / per-queue / matching condition flags (DEST / DEST_CHANNEL / TIME_BEFORE / TIME_AFTER / EVENT_TYPE / IGNORE_OFF / TAG_MATCH).
- snd_seq_queue_remove_cells + snd_seq_prioq_remove_events implement.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `client_number_in_range` | INVARIANT | per-client: number ∈ [0, SNDRV_SEQ_MAX_CLIENTS=192). |
| `port_number_in_range` | INVARIANT | per-port: addr.port ∈ [0, SNDRV_SEQ_MAX_PORTS=254). |
| `queue_id_in_range` | INVARIANT | per-queue: queue ∈ [0, SNDRV_SEQ_MAX_QUEUES=32). |
| `hop_bounded` | INVARIANT | per-dispatch: hop ≤ SNDRV_SEQ_MAX_HOPS=8 — else EINVAL. |
| `pool_counter_consistent` | INVARIANT | per-pool: counter ≤ total_elements; free-list length = total_elements - counter. |
| `prioq_time_sorted` | INVARIANT | per-prioq: traversal head→tail yields monotonic non-decreasing time. |
| `subscribers_refcount_balanced` | INVARIANT | per-subs: ref_count = (in src.list_head ? 1 : 0) + (in dest.list_head ? 1 : 0). |
| `use_lock_drained_at_free` | INVARIANT | per-{client, port, queue, pool}-free: snd_use_lock_sync runs to 0. |
| `event_length_flag_matches` | INVARIANT | per-event: FIXED vs VARIABLE vs VARUSR flag matches actual chained-cell payload. |
| `dest_address_resolved_pre_deliver` | INVARIANT | per-deliver_single: dest client + dest port both refcounted before fop. |

### Layer 2: TLA+

`sound/core/seq.tla`:
- Per-state-machine: client { NO_CLIENT, USER, KERNEL } × queue { STOPPED, RUNNING } × event { ENQUEUED, IN_PRIOQ, DELIVERED, FREED }.
- Properties:
  - `safety_no_event_lost_in_pool` — per-cell: every snd_seq_event_dup eventually snd_seq_cell_free.
  - `safety_hop_count_terminates_loops` — per-dispatch: subscription cycles bounded by MAX_HOPS=8.
  - `safety_no_use_after_free_via_use_lock` — per-ref: snd_use_lock_sync at free fences readers.
  - `safety_announce_sent_on_lifecycle` — per-client open / close + port add / del + connect / disconnect → announce port event.
  - `safety_prioq_fifo_within_same_time` — per-prioq: equal-timestamp cells preserve insertion order (stable sort).
  - `liveness_queue_drains_when_running` — per-running-queue: ready events eventually dispatched on tick.
  - `liveness_writer_unblocks_on_pool_free` — per-pool-full + write-blocked: cell_free → wake.
  - `safety_pool_closing_no_new_alloc` — per-pool: closing flag set ⟹ subsequent event_dup fails.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SndSeqClient::new` post: number ∈ valid range; type = USER ∨ KERNEL; fifo iff input ∧ USER | `SndSeqClient::new` |
| `SndSeqClient::dispatch_event` post: cell freed; hop bumped on subscribe path | `SndSeqClient::dispatch_event` |
| `SndSeqClient::deliver_to_subscribers` post: 1 event per subscriber in c_src.list_head | `SndSeqClient::deliver_to_subscribers` |
| `SndSeqClientPort::connect` post: subscribers in src.c_src.list_head ∧ dest.c_dest.list_head | `SndSeqClientPort::connect` |
| `SndSeqClientPort::disconnect` post: subscribers removed from both lists; ref_count = 0 ⟹ freed | `SndSeqClientPort::disconnect` |
| `SndSeqQueue::enqueue_event` post: cell on tickq xor timeq per flags | `SndSeqQueue::enqueue_event` |
| `SndSeqQueue::check` post: prioq head time > cur_time (else dispatched) | `SndSeqQueue::check` |
| `SndSeqPrioq::cell_in` post: list time-sorted | `SndSeqPrioq::cell_in` |
| `SndSeqPool::event_dup` post: cellp set; pool.counter += 1 (+chain for variable) | `SndSeqPool::event_dup` |
| `SndSeqPool::cell_free` post: pool.counter -= 1; wake output_sleep | `SndSeqPool::cell_free` |
| `SndSeqTimer::interrupt` post: tmr.cur_time + tmr.ticks advanced; queue check invoked | `SndSeqTimer::interrupt` |

### Layer 4: Verus/Creusot functional

`Per-snd_seq_open → snd_seq_create_port → snd_seq_port_connect (src.SUBS_READ × dest.SUBS_WRITE) → snd_seq_write → snd_seq_event_dup → snd_seq_enqueue_event (TIME_STAMP_TICK | _REAL → tickq | timeq) → snd_seq_timer_interrupt → snd_seq_check_queue → snd_seq_dispatch_event → deliver_to_subscribers → __snd_seq_deliver_single_event (USER→fifo | KERNEL→event_input) → snd_seq_read → snd_seq_cell_free → snd_seq_port_disconnect → snd_seq_release` semantic equivalence: per-`Documentation/sound/designs/seq-oss.rst` and ALSA UAPI semantics in `include/uapi/sound/asequencer.h`. Subscription fan-out semantics tested against `aplaymidi`/`arecordmidi`/`amidi -d`/`jackd MIDI bridge` traces.

### hardening

(Inherits row-1 features from `sound/00-overview.md` § Hardening.)

ALSA-sequencer reinforcement:

- **Per-SNDRV_SEQ_MAX_HOPS = 8 cap** — defense against per-subscription-cycle infinite-loop.
- **Per-SNDRV_SEQ_MAX_CLIENTS = 192 cap** — defense against per-client-table exhaustion.
- **Per-SNDRV_SEQ_MAX_PORTS = 254 cap per client** — defense against per-port-flood.
- **Per-SNDRV_SEQ_MAX_QUEUES = 32 cap** — defense against per-queue-table exhaustion.
- **Per-SNDRV_SEQ_MAX_EVENTS = 2000 per-client pool cap** — defense against per-pool memory exhaustion.
- **Per-snd_use_lock_sync drain at free** — defense against per-UAF in concurrent dispatch.
- **Per-client_init_data zaps clienttab on init** — defense against per-stale-slot reuse on module reload.
- **Per-check_subscription_permission gating SUBS_READ × SUBS_WRITE** — defense against per-unauthorized fan-out.
- **Per-pool->closing fence** — defense against per-dup-after-close race.
- **Per-fifo overflow atomic + -ENOSPC** — defense against per-silent-drop.
- **Per-bounce_error_event opt-in via FILTER_BOUNCE** — defense against per-bounce-storm.
- **Per-queue->locked enforcement** — defense against per-non-owner timer manipulation.
- **Per-snd_seq_kernel_client_ioctl bounded to OSS path** — defense against per-arbitrary-ioctl from kernel client.
- **Per-event_filter bitmap gating types** — defense against per-event-flood / per-type DoS.
- **Per-VARUSR copy_from_user staging** — defense against per-userspace-pointer-trust during dispatch.
- **Per-system-announce port read-only (NO_EXPORT)** — defense against per-spoofed announce.

