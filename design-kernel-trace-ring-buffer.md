---
title: "Tier-3: kernel/trace/ring_buffer.c — Lock-free tracing ring buffer"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **tracing ring buffer** is a per-CPU, lock-free, multi-producer / single-reader event log used by ftrace, perf-trace and bpf-trace. Per-CPU `struct ring_buffer_per_cpu` owns a circular list of sub-buffer **pages** (`struct buffer_page`) wrapping a `struct buffer_data_page` of order `subbuf_order`. Per-CPU writers reserve event-bytes via `ring_buffer_lock_reserve()` → `__rb_reserve_next()` using `local_add_return` on `tail_page->write` (no IRQ disable beyond preempt-disable + recursion-bit). Per-page-boundary crossings are arbitrated via `rb_move_tail()` + `rb_handle_head_page()`, which use **cmpxchg on `list->next` low bits**: encoding `RB_PAGE_HEAD` (1), `RB_PAGE_UPDATE` (2), `RB_PAGE_NORMAL` (0), `RB_PAGE_MOVED` (4). Per-reader path uses `arch_spinlock_t` and **swaps in `reader_page`** for the head via `rb_head_page_replace()` cmpxchg — readers and writers cooperate without blocking each other. Per-overrun (overwrite mode) the writer takes ownership of the head's `entries` count into `cpu_buffer->overrun`; in non-overwrite mode `dropped_events` increments. Per-`lost_events` accounting is propagated to readers across reader-page swaps via `last_overrun` deltas. Per-snapshot a buffer can be swapped CPU-for-CPU via `ring_buffer_swap_cpu()`. Per-`ring_buffer_resize()` adds/removes pages while writers are live by deferring work through `update_pages_handler`. Critical for: tracing-correctness under NMI / IRQ / preempt nesting, no-deadlock-on-trace, no-drop-on-recursion.

This Tier-3 covers `kernel/trace/ring_buffer.c` (~8103 lines).

### Acceptance Criteria

- [ ] AC-1: 32-byte event write/read round-trip preserves payload across pages.
- [ ] AC-2: Concurrent N-CPU producers (one per CPU) commit without lock contention or lost commits.
- [ ] AC-3: Overwrite mode + buffer full: oldest events overrun; overrun++ matches; entries_bytes consistent.
- [ ] AC-4: Non-overwrite mode + full: lock_reserve returns NULL; dropped_events++.
- [ ] AC-5: Interrupted reservation (IRQ inside write) — both events commit in order; no payload corruption.
- [ ] AC-6: NMI-context reservation on !ARCH_HAVE_NMI_SAFE_CMPXCHG: returns NULL; recorder does not deadlock.
- [ ] AC-7: Reader peek/consume returns events in commit order; consume advances reader_page when page drained.
- [ ] AC-8: lost_events propagated from overrun count delta on reader-page swap.
- [ ] AC-9: ring_buffer_resize grow/shrink while writers live preserves committed events ≤ min(old_size, new_size).
- [ ] AC-10: ring_buffer_swap_cpu(a, b, cpu): swaps per-CPU buffers; trace_buffer back-pointers updated.
- [ ] AC-11: record_disable / record_enable: pending reserve completes, no new reserves after disable.
- [ ] AC-12: trace_recursive_lock detects recursion within same context-bit (NMI/IRQ/SOFTIRQ/NORMAL).
- [ ] AC-13: rb_head_page_set cmpxchg: HEAD→UPDATE only one writer succeeds; others see MOVED/NORMAL.
- [ ] AC-14: ring_buffer_read_start iterator + advance walks all events as of start; misses events committed later.
- [ ] AC-15: subbuf_order_set: re-allocates pages with new order; subbuf_size updated.

### Architecture

```
struct RingBufferPerCpu {
  cpu: i32,
  record_disabled: AtomicU32,
  resize_disabled: AtomicU32,
  buffer: *TraceBuffer,
  reader_lock: RawSpinLock,
  lock: ArchSpinLock,                    // internal head-swap
  free_page: Option<*BufferDataPage>,
  nr_pages: u64,
  current_context: u32,                  // recursion bitmap
  pages: *ListHead,                      // circular list of buffer_page
  cnt: u64,                              // pages generation counter
  head_page: *BufferPage,                // read from head
  tail_page: *BufferPage,                // write to tail
  commit_page: *BufferPage,              // committed
  reader_page: *BufferPage,              // private to reader
  lost_events: u64,
  last_overrun: u64,
  nest: u64,
  entries_bytes: LocalT,
  entries: LocalT,
  overrun: LocalT,
  commit_overrun: LocalT,
  dropped_events: LocalT,
  committing: LocalT,
  commits: LocalT,
  pages_touched: LocalT,
  pages_lost: LocalT,
  pages_read: LocalT,
  last_pages_touch: i64,
  shortest_full: usize,
  read: u64,
  read_bytes: u64,
  write_stamp: RbTime,                   // last completed event ts
  before_stamp: RbTime,                  // tentative ts (interrupt-safe)
  event_stamp: [u64; MAX_NEST],
  read_stamp: u64,
  pages_removed: u64,
  mapped: u32,
  user_mapped: u32,
  mapping_lock: Mutex,
  subbuf_ids: *Box<[*BufferPage]>,
  meta_page: *TraceBufferMeta,
  ring_meta: *RingBufferCpuMeta,
  remote: *RingBufferRemote,
  nr_pages_to_update: i64,
  new_pages: ListHead,
  update_pages_work: WorkStruct,
  update_done: Completion,
  irq_work: RbIrqWork,
}

struct BufferPage {
  list: ListHead,                        // low bits: RB_PAGE_*
  write: LocalT,                         // index of next write (low 20) + INTCNT (high 12)
  read: u32,                             // index of next read
  entries: LocalT,
  real_end: u64,
  order: u32,
  id: u32,                               // 30 bits
  range: u32,                            // 1 bit
  page: *BufferDataPage,
}

struct RingBufferEvent {
  type_len: u32,                         // 5 bits
  time_delta: u32,                       // 27 bits
  array: [u32; 0],                       // variable payload
}

struct RingBufferIter {
  cpu_buffer: *RingBufferPerCpu,
  head: u64,
  next_event: u64,
  head_page: *BufferPage,
  cache_reader_page: *BufferPage,
  cache_read: u64,
  cache_pages_removed: u64,
  read_stamp: u64,
  page_stamp: u64,
  event: *RingBufferEvent,
  event_size: usize,
  missed_events: i32,
}

const RB_PAGE_NORMAL: u64 = 0;
const RB_PAGE_HEAD:   u64 = 1;
const RB_PAGE_UPDATE: u64 = 2;
const RB_PAGE_MOVED:  u64 = 4;          // not in mask
const RB_FLAG_MASK:   u64 = 3;
const RB_WRITE_MASK:   u32 = 0x000F_FFFF;
const RB_WRITE_INTCNT: u32 = 1 << 20;
```

`RingBuffer::lock_reserve(self, length) -> *RingBufferEvent`:
1. preempt_disable_notrace.
2. cpu = smp_processor_id.
3. if !self.cpumask.test(cpu): return NULL.
4. cpu_buffer = self.buffers[cpu].
5. if self.record_disabled.load > 0 ∨ cpu_buffer.record_disabled.load > 0: NULL.
6. if cpu_buffer.recursive_lock(): NULL.
7. event = cpu_buffer.reserve_next_event(self, length).
8. if !event: cpu_buffer.recursive_unlock; preempt_enable_notrace; NULL.
9. return event.  /* preempt remains disabled until unlock_commit */

`RingBufferPerCpu::reserve_next_event(buffer, length) -> *RingBufferEvent`:
1. if in_nmi ∧ !ARCH_HAVE_NMI_SAFE_CMPXCHG: NULL.
2. self.start_commit  /* committing+=1 */
3. info.length = rb_calculate_event_length(length).
4. if buffer.time_stamp_abs: add_ts_default = RB_ADD_STAMP_ABSOLUTE; info.length += RB_LEN_TIME_EXTEND.
5. again (nr_loops < 3):
   - info.add_timestamp = add_ts_default.
   - event = self.__reserve_next(&info).
   - if IS_ERR(event) ∧ -EAGAIN: goto again.
6. if !event: out_fail (end_commit; return NULL).
7. return event.

`RingBufferPerCpu::__reserve_next(self, info) -> *RingBufferEvent`:
1. tail_page = READ_ONCE(self.tail_page); info.tail_page = tail_page.
2. /* A */ w = local_read(&tail_page.write) & RB_WRITE_MASK; barrier.
3. rb_time_read(&self.before_stamp, &info.before); rb_time_read(&self.write_stamp, &info.after); info.ts = rb_time_stamp(self.buffer).
4. /* compute info.delta */
5. /* B */ rb_time_set(&self.before_stamp, info.ts).
6. /* C */ write = local_add_return(info.length, &tail_page.write); write &= RB_WRITE_MASK.
7. tail = write - info.length.
8. if write > self.buffer.subbuf_size: check_buffer(CHECK_FULL_PAGE); return self.move_tail(tail, info).
9. if tail == w:  /* fast path */
   - /* D */ rb_time_set(&self.write_stamp, info.ts).
   - info.delta = info.ts - info.after.
   - check_buffer(tail).
10. else: /* slow path - interrupted between A and C */
    - rb_time_read(&before_stamp, &info.before); ts = rb_time_stamp; rb_time_set(&before_stamp, ts).
    - /* E */ rb_time_read(&write_stamp, &info.after); barrier.
    - /* F */ if write == (local_read(&tail_page.write) & RB_WRITE_MASK) ∧ info.after == info.before ∧ info.after < ts: info.delta = ts - info.after; else info.delta = 0.
    - info.ts = ts.
11. event = __rb_page_index(tail_page, tail); rb_update_event(event, info).
12. local_inc(&tail_page.entries).
13. if !tail: tail_page.page.time_stamp = info.ts.
14. local_add(info.length, &self.entries_bytes).
15. return event.

`RingBufferPerCpu::move_tail(self, tail, info) -> *RingBufferEvent`:
1. next_page = info.tail_page; rb_inc_page(&next_page).
2. if next_page == self.commit_page: local_inc(&self.commit_overrun); out_reset (NULL).
3. if rb_is_head_page(next_page, &info.tail_page.list):
   - if !rb_is_reader_page(self.commit_page):
     - if !(self.buffer.flags & RB_FL_OVERWRITE): local_inc(&self.dropped_events); out_reset.
     - ret = self.handle_head_page(info.tail_page, next_page).
     - if ret < 0: out_reset; if ret > 0: out_again.
   - else: /* commit on reader_page; small-buffer corner */
     - if commit_page != tail_page ∧ commit_page == reader_page: local_inc(&commit_overrun); out_reset.
4. self.tail_page_update(info.tail_page, next_page).
5. out_again: self.reset_tail(tail, info); self.end_commit; local_inc(&self.committing); return Err(-EAGAIN).
6. out_reset: self.reset_tail(tail, info); return NULL.

`RingBufferPerCpu::handle_head_page(self, tail_page, next_page) -> i32`:
1. entries = rb_page_entries(next_page).
2. type = rb_head_page_set_update(self, next_page, tail_page, RB_PAGE_HEAD).
   /* cmpxchg list.next: HEAD→UPDATE */
3. switch type:
   - RB_PAGE_HEAD: local_add(entries, &self.overrun); local_sub(rb_page_commit(next_page), &self.entries_bytes); local_inc(&self.pages_lost); if ring_meta: rb_update_meta_head.
   - RB_PAGE_UPDATE: /* interrupted mid-update; fall through */
   - RB_PAGE_NORMAL: return 1  /* interrupt finished it */
   - RB_PAGE_MOVED: return 1  /* reader took it */
   - default: RB_WARN_ON; return -1.
4. new_head = next_page; rb_inc_page(&new_head).
5. ret = rb_head_page_set_head(self, new_head, next_page, RB_PAGE_NORMAL).
6. if ret == RB_PAGE_NORMAL ∧ READ_ONCE(self.tail_page) != tail_page ∧ != next_page:
   - rb_head_page_set_normal(self, new_head, next_page, RB_PAGE_HEAD).
7. if type == RB_PAGE_HEAD:
   - ret = rb_head_page_set_normal(self, next_page, tail_page, RB_PAGE_UPDATE).
   - RB_WARN_ON(ret != RB_PAGE_UPDATE).
8. return 0.

`BufferPage::head_set(head, prev, old_flag, new_flag) -> i32`:
1. val = (u64)&head.list & ~RB_FLAG_MASK.
2. ret = cmpxchg(&prev.list.next, val | old_flag, val | new_flag).
3. if (ret & ~RB_FLAG_MASK) != val: return RB_PAGE_MOVED.
4. return ret & RB_FLAG_MASK.

`RingBufferPerCpu::get_reader_page(self) -> *BufferPage`:
1. local_irq_save; arch_spin_lock(&self.lock).
2. again (nr_loops < 3):
   - reader = self.reader_page.
   - if reader.read < rb_page_size(reader): goto out (more on reader).
   - if self.commit_page == self.reader_page ∨ rb_num_of_entries(self) == 0: reader = NULL; goto out.
   - local_set(&reader.write, 0); local_set(&reader.entries, 0); reader.real_end = 0.
   - spin:
     - reader = self.set_head_page.
     - if !reader: goto out.
     - self.reader_page.list.next = rb_list_head(reader.list.next).
     - self.reader_page.list.prev = reader.list.prev.
     - self.pages = reader.list.prev.
     - rb_set_list_to_head(&self.reader_page.list).
     - smp_mb; overwrite = local_read(&self.overrun).
     - if !rb_head_page_replace(reader, self.reader_page): goto spin.
   - if self.ring_meta: rb_update_meta_reader(self, reader).
   - rb_list_head(reader.list.next).prev = &self.reader_page.list; rb_inc_page(&self.head_page).
   - self.cnt += 1; local_inc(&self.pages_read).
   - self.reader_page = reader; self.reader_page.read = 0.
   - if overwrite != self.last_overrun: self.lost_events = overwrite - self.last_overrun; self.last_overrun = overwrite.
   - goto again.
3. out:
   - if reader ∧ reader.read == 0: self.read_stamp = reader.page.time_stamp.
   - arch_spin_unlock(&self.lock); local_irq_restore.
4. /* Wait up to 1s for in-flight writer to clear write past page boundary */
5. for nr_loops in 0..1_000_000: if !reader ∨ rb_page_write(reader) <= bsize: break; udelay(1); smp_rmb.
6. smp_rmb; return reader.

`RingBuffer::peek(self, cpu, ts, lost_events) -> *RingBufferEvent`:
1. cpu_buffer = self.buffers[cpu].
2. raw_spin_lock_irqsave(&cpu_buffer.reader_lock).
3. event = rb_buffer_peek(cpu_buffer, ts, lost_events).
4. raw_spin_unlock_irqrestore.
5. return event.

`RingBuffer::consume(self, cpu, ts, lost_events) -> *RingBufferEvent`:
1. preempt_disable.
2. if !self.cpumask.test(cpu): event = NULL; goto out.
3. cpu_buffer = self.buffers[cpu]; raw_spin_lock_irqsave(&cpu_buffer.reader_lock).
4. event = rb_buffer_peek(cpu_buffer, ts, lost_events).
5. if event ∧ event.type_len != RINGBUF_TYPE_PADDING: cpu_buffer.read += 1; rb_advance_reader(cpu_buffer).
6. raw_spin_unlock_irqrestore; preempt_enable.
7. return event.

`RingBuffer::swap_cpu(self, other, cpu) -> i32`:
1. if !cpumask_test_cpu(cpu, self.cpumask) ∨ !cpumask_test_cpu(cpu, other.cpumask): -EINVAL.
2. if self.subbuf_size != other.subbuf_size: -EINVAL.
3. ours = self.buffers[cpu]; theirs = other.buffers[cpu].
4. /* must be empty of in-flight reservations */
5. atomic_inc(&self.record_disabled); atomic_inc(&other.record_disabled).
6. /* race against ring_buffer_lock_reserve */
7. if local_read(&ours.committing) ∨ local_read(&theirs.committing): err = -EBUSY; goto out.
8. self.buffers[cpu] = theirs; other.buffers[cpu] = ours.
9. ours.buffer = other; theirs.buffer = self.
10. out: atomic_dec(&self.record_disabled); atomic_dec(&other.record_disabled); return err.

`RingBuffer::resize(self, size, cpu_id) -> i32`:
1. if !self: return 0.
2. if cpu_id != RING_BUFFER_ALL_CPUS ∧ !self.cpumask.test(cpu_id): return 0.
3. nr_pages = DIV_ROUND_UP(size, self.subbuf_size); if < 2: nr_pages = 2.
4. guard(cpus_read_lock); mutex_lock(&self.mutex); atomic_inc(&self.resizing).
5. if cpu_id == RING_BUFFER_ALL_CPUS:
   - For each cpu: if resize_disabled: err = -EBUSY.
   - For each cpu: nr_pages_to_update = nr_pages - nr; if > 0: __rb_allocate_pages into new_pages.
   - For each cpu (online): schedule_work_on(cpu, &update_pages_work) or run inline.
   - For each cpu: wait_for_completion(&update_done).
6. else: single-cpu path same shape.
7. if record_disabled: atomic_inc(&record_disabled); synchronize_rcu; rb_check_pages each cpu; atomic_dec.
8. atomic_dec(&resizing); mutex_unlock; return 0.

### Out of Scope

- kernel/trace/trace.c front-end (covered in `trace.md` Tier-3 if expanded)
- kernel/trace/ftrace.c function tracer (covered in `ftrace.md` Tier-3 if expanded)
- kernel/trace/trace_events.c event subsystem (covered in `trace-events.md` Tier-3 if expanded)
- kernel/bpf/ringbuf.c BPF ring buffer (separate Tier-3 if expanded)
- perf_event ring buffer (covered in `events/core.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct trace_buffer` | per-buffer container (all CPUs) | `TraceBuffer` |
| `struct ring_buffer_per_cpu` | per-CPU lock-free ring | `RingBufferPerCpu` |
| `struct buffer_page` | per-sub-buffer descriptor | `BufferPage` |
| `struct buffer_data_page` | per-page raw event-storage header | `BufferDataPage` |
| `struct ring_buffer_event` | per-event header (type_len, time_delta) | `RingBufferEvent` |
| `struct ring_buffer_iter` | per-reader iterator state | `RingBufferIter` |
| `struct rb_event_info` | per-reserve transient info | `RbEventInfo` |
| `__ring_buffer_alloc()` | per-allocate trace_buffer | `RingBuffer::alloc` |
| `ring_buffer_free()` | per-free trace_buffer | `RingBuffer::free` |
| `ring_buffer_resize()` | per-grow / shrink pages | `RingBuffer::resize` |
| `ring_buffer_lock_reserve()` | per-reserve event-bytes | `RingBuffer::lock_reserve` |
| `ring_buffer_unlock_commit()` | per-commit reserved event | `RingBuffer::unlock_commit` |
| `ring_buffer_write()` | per-write small payload (atomic) | `RingBuffer::write` |
| `ring_buffer_discard_commit()` | per-discard reserved | `RingBuffer::discard_commit` |
| `__rb_reserve_next()` | per-CPU reserve core | `RingBufferPerCpu::reserve_next` |
| `rb_move_tail()` | per-page boundary crossing | `RingBufferPerCpu::move_tail` |
| `rb_handle_head_page()` | per-overrun (writer hit head) | `RingBufferPerCpu::handle_head_page` |
| `rb_head_page_set` / `_set_head` / `_set_update` / `_set_normal` | per-cmpxchg flag transition | `BufferPage::set_*` |
| `rb_set_head_page()` | per-find current head | `RingBufferPerCpu::set_head_page` |
| `rb_head_page_replace()` | per-swap reader for head | `BufferPage::head_replace` |
| `rb_tail_page_update()` | per-advance tail pointer | `RingBufferPerCpu::tail_page_update` |
| `__rb_get_reader_page()` | per-reader-page swap | `RingBufferPerCpu::get_reader_page` |
| `ring_buffer_peek()` | per-peek next event (no consume) | `RingBuffer::peek` |
| `ring_buffer_consume()` | per-read next event (consume) | `RingBuffer::consume` |
| `ring_buffer_iter_peek()` | per-iter non-consuming peek | `RingBufferIter::peek` |
| `ring_buffer_read_start()` / `_finish()` | per-iterator life-cycle | `RingBufferIter::start` / `finish` |
| `ring_buffer_iter_advance()` | per-iter advance | `RingBufferIter::advance` |
| `ring_buffer_iter_empty()` | per-iter-empty test | `RingBufferIter::is_empty` |
| `ring_buffer_swap_cpu()` | per-snapshot swap | `RingBuffer::swap_cpu` |
| `ring_buffer_record_disable()` / `_enable()` | per-suspend writers | `RingBuffer::record_disable` / `enable` |
| `ring_buffer_record_off()` / `_on()` | per-master switch | `RingBuffer::record_off` / `on` |
| `ring_buffer_reset_cpu()` / `_reset()` | per-reset state | `RingBuffer::reset_cpu` / `reset` |
| `ring_buffer_read_page()` | per-splice-page-to-user | `RingBuffer::read_page` |
| `ring_buffer_subbuf_order_set()` | per-set sub-buffer order | `RingBuffer::subbuf_order_set` |
| `ring_buffer_nest_start()` / `_nest_end()` | per-allow nested trace | `RingBuffer::nest_start` / `nest_end` |
| `trace_recursive_lock` / `_unlock` | per-CTX recursion guard | `RingBufferPerCpu::recursive_lock` / `unlock` |
| `rb_time_stamp()` | per-clock read | `RingBuffer::time_stamp` |
| `rb_wakeups()` | per-reader-wake on commit | `RingBufferPerCpu::wakeups` |
| `ring_buffer_wait()` / `_poll_wait()` | per-reader wait/poll | `RingBuffer::wait` / `poll_wait` |
| `RB_PAGE_HEAD / _UPDATE / _NORMAL / _MOVED` | per-list-bit flags | `RbPageFlag` |
| `RB_WRITE_MASK / RB_WRITE_INTCNT` | per-write counter mask | `RbWrite` |

### compatibility contract

REQ-1: struct ring_buffer_per_cpu:
- cpu: per-CPU index.
- record_disabled / resize_disabled: per-suspend atomic counters.
- buffer: per-back-pointer to trace_buffer.
- reader_lock: raw_spinlock serializing readers.
- lock: arch_spinlock_t for internal head-swap.
- pages: per-circular-list head.
- head_page / tail_page / commit_page / reader_page: per-pointer-quad.
- lost_events / last_overrun: per-overrun delta tracking.
- nest: per-context-nest depth.
- entries_bytes / entries / overrun / commit_overrun / dropped_events / committing / commits / pages_touched / pages_lost / pages_read: per-local_t stats.
- write_stamp / before_stamp / read_stamp / event_stamp[MAX_NEST]: per-timestamp shadows.
- nr_pages / pages_removed: per-page-count.
- irq_work: per-reader wake mechanism.

REQ-2: struct buffer_page:
- list (`struct list_head`): per-circular-list link.
- write (`local_t`): per-byte-offset of next reserve (low 20 bits) + interrupt-count (high 12 bits, RB_WRITE_INTCNT).
- read: per-reader byte-offset.
- entries (`local_t`): per-event-count.
- real_end: per-tail-padded byte offset.
- order: per-page-order (matches buffer->subbuf_order).
- id:30 + range:1: per-ID for external mapping.
- page: per-pointer to buffer_data_page (the raw event area).
- /* The buffer_page list-pointer carries low bits as flags */
- list.next ↑ low bits: RB_PAGE_NORMAL (0) / _HEAD (1) / _UPDATE (2).

REQ-3: struct ring_buffer_event:
- time_delta: per-delta from previous event (27 bits).
- type_len: per-event-type (0 = padding, 1 = TIME_EXTEND, 2 = TIME_STAMP, ≥3 = data of fixed/variable length).
- array[]: per-payload.

REQ-4: struct ring_buffer_iter:
- cpu_buffer: per-back-pointer.
- head_page: per-cached head.
- head / next_event: per-byte-offsets.
- cache_reader_page / cache_read / cache_pages_removed: per-detect reader-side mutation.
- read_stamp / page_stamp: per-cumulative timestamp.
- event / event_size: per-event-being-read.
- missed_events: per-lost-events seen.

REQ-5: __ring_buffer_alloc(size, flags, key):
- buffer = kzalloc(sizeof(*buffer)).
- buffer.flags = flags (RB_FL_OVERWRITE possibly).
- nr_pages = DIV_ROUND_UP(size, subbuf_size); minimum 2.
- For each possible CPU (cpus_read_lock): allocate ring_buffer_per_cpu + nr_pages buffer_page.
- Each cpu_buffer: reader_page + nr_pages circular list of buffer_page.
- Activate head: rb_head_page_activate sets RB_PAGE_HEAD on prev(head_page).list.next.
- Register cpu-hotplug node.

REQ-6: ring_buffer_lock_reserve(buffer, length) -> event:
- preempt_disable_notrace.
- if !cpumask_test_cpu(cpu, buffer.cpumask): return NULL.
- cpu_buffer = buffer.buffers[cpu].
- if atomic_read(&buffer.record_disabled) ∨ atomic_read(&cpu_buffer.record_disabled): return NULL.
- if trace_recursive_lock(cpu_buffer): return NULL.
- event = rb_reserve_next_event(buffer, cpu_buffer, length).
- if !event: trace_recursive_unlock; preempt_enable_notrace; return NULL.
- return event.  /* preempt remains disabled until unlock_commit */

REQ-7: rb_reserve_next_event(buffer, cpu_buffer, length):
- if in_nmi ∧ !ARCH_HAVE_NMI_SAFE_CMPXCHG: return NULL.
- rb_start_commit(cpu_buffer)  /* committing+=1, commits-incremented at end */
- length = rb_calculate_event_length(length)  /* round up, header */
- Bounded retry (nr_loops < 3, RB_WARN_ON):
  - event = __rb_reserve_next(cpu_buffer, &info).
  - if IS_ERR(event) ∧ PTR_ERR(event) == -EAGAIN: goto again.
- if !event: out_fail (rb_end_commit, return NULL).
- return event.

REQ-8: __rb_reserve_next(cpu_buffer, info):
- /* A */ tail_page = READ_ONCE(cpu_buffer.tail_page); w = tail_page.write & RB_WRITE_MASK.
- rb_time_read(&before_stamp, &info.before); rb_time_read(&write_stamp, &info.after); info.ts = rb_time_stamp.
- if w == 0: info.delta = 0  /* page-first event */
- else if info.before != info.after: force absolute timestamp (add_timestamp |= RB_ADD_STAMP_FORCE|RB_ADD_STAMP_EXTEND).
- else: info.delta = info.ts - info.after.
- /* B */ rb_time_set(&before_stamp, info.ts).
- /* C */ write = local_add_return(info.length, &tail_page.write); write &= RB_WRITE_MASK.
- tail = write - info.length.
- if write > subbuf_size: rb_move_tail(...)  /* page-crossing slow-path */
- if tail == w: /* fast path - nothing interrupted */
  - /* D */ rb_time_set(&write_stamp, info.ts).
- else: /* slow path interrupted between A and C - recompute delta */
- event = __rb_page_index(tail_page, tail); rb_update_event(event, info).
- local_inc(&tail_page.entries).
- if !tail: tail_page.page.time_stamp = info.ts.
- local_add(info.length, &cpu_buffer.entries_bytes).
- return event.

REQ-9: rb_move_tail(cpu_buffer, tail, info):
- /* Writer crossed a page boundary - need to advance tail */
- next_page = tail_page; rb_inc_page(&next_page).
- if next_page == commit_page: commit_overrun++; out_reset (return NULL).
- if rb_is_head_page(next_page, &tail_page.list):
  - /* Buffer is full at this page boundary */
  - if !rb_is_reader_page(commit_page):
    - if !(buffer.flags & RB_FL_OVERWRITE): dropped_events++; out_reset.
    - ret = rb_handle_head_page(cpu_buffer, tail_page, next_page).
    - if ret < 0: out_reset; if ret > 0: out_again (retry).
  - else: /* commit on reader_page; handle small-buffer edge */
- rb_tail_page_update(cpu_buffer, tail_page, next_page).
- out_again: rb_reset_tail(...); rb_end_commit (decrement committing); local_inc(committing); return ERR_PTR(-EAGAIN).
- out_reset: rb_reset_tail; return NULL.

REQ-10: rb_handle_head_page(cpu_buffer, tail_page, next_page):
- entries = rb_page_entries(next_page).
- type = rb_head_page_set_update(cpu_buffer, next_page, tail_page, RB_PAGE_HEAD).
  /* cmpxchg list.next: HEAD -> UPDATE; result in {HEAD, UPDATE, NORMAL, MOVED} */
- switch (type):
  - RB_PAGE_HEAD: /* we won the race; account overrun */
    - local_add(entries, &overrun); local_sub(rb_page_commit(next_page), &entries_bytes); local_inc(&pages_lost).
    - if ring_meta: rb_update_meta_head.
  - RB_PAGE_UPDATE: /* nested interrupt observed mid-update; continue */
  - RB_PAGE_NORMAL: /* interrupt already finished move */ return 1.
  - RB_PAGE_MOVED: /* reader on other CPU swapped */ return 1.
- new_head = next_page; rb_inc_page(&new_head).
- ret = rb_head_page_set_head(cpu_buffer, new_head, next_page, RB_PAGE_NORMAL).
- if ret == RB_PAGE_NORMAL ∧ READ_ONCE(tail_page) != tail_page ∧ != next_page:
  - rb_head_page_set_normal(new_head, next_page, RB_PAGE_HEAD).
- if type == RB_PAGE_HEAD: ret = rb_head_page_set_normal(next_page, tail_page, RB_PAGE_UPDATE); RB_WARN_ON(ret != RB_PAGE_UPDATE).
- return 0.

REQ-11: rb_head_page_set(cpu_buffer, head, prev, old_flag, new_flag):
- /* cmpxchg on prev.list.next: encode head pointer with flag bits */
- val = (unsigned long)&head.list & ~RB_FLAG_MASK.
- ret = cmpxchg(&prev.list.next, val | old_flag, val | new_flag).
- if (ret & ~RB_FLAG_MASK) != val: return RB_PAGE_MOVED.
- return ret & RB_FLAG_MASK.

REQ-12: __rb_get_reader_page(cpu_buffer):
- /* Reader-page swap algorithm */
- local_irq_save; arch_spin_lock(&cpu_buffer.lock).
- again: if reader_page.read < rb_page_size(reader_page): return reader_page.
- if commit_page == reader_page ∨ rb_num_of_entries == 0: return NULL.
- /* Reset reader_page to empty, splice into list at head */
- local_set(&reader_page.write, 0); local_set(&reader_page.entries, 0); reader_page.real_end = 0.
- spin: reader = rb_set_head_page(cpu_buffer).
- reader_page.list.next = rb_list_head(reader.list.next); reader_page.list.prev = reader.list.prev.
- cpu_buffer.pages = reader.list.prev.
- rb_set_list_to_head(&reader_page.list).
- smp_mb; overwrite = local_read(&overrun).
- /* cmpxchg HEAD bit: rb_head_page_replace */
- ret = rb_head_page_replace(reader, reader_page).
- if !ret: goto spin (writer racing).
- rb_list_head(reader.list.next).prev = &reader_page.list; rb_inc_page(&head_page).
- cnt++; local_inc(&pages_read).
- reader_page = reader; reader_page.read = 0.
- if overwrite != last_overrun: lost_events = overwrite - last_overrun; last_overrun = overwrite.
- goto again.
- /* Wait up to ~1s for an in-flight writer past page boundary */
- for nr_loops in 0..USECS_WAIT: if rb_page_write(reader) <= bsize: break; udelay(1); smp_rmb.
- smp_rmb. arch_spin_unlock; local_irq_restore. return reader.

REQ-13: ring_buffer_unlock_commit(buffer):
- cpu = smp_processor_id; cpu_buffer = buffer.buffers[cpu].
- rb_commit(cpu_buffer)  /* local_inc(&commits) */
- rb_wakeups(buffer, cpu_buffer)  /* irq_work queue if waiters */
- trace_recursive_unlock(cpu_buffer); preempt_enable_notrace.
- return 0.

REQ-14: ring_buffer_write(buffer, length, data):
- event = ring_buffer_lock_reserve(buffer, length).
- if !event: return -EBUSY.
- memcpy(ring_buffer_event_data(event), data, length).
- ring_buffer_unlock_commit(buffer).
- return 0.

REQ-15: ring_buffer_peek(buffer, cpu, ts, lost_events):
- cpu_buffer = buffer.buffers[cpu].
- raw_spin_lock_irqsave(&cpu_buffer.reader_lock).
- event = rb_buffer_peek(cpu_buffer, ts, lost_events).
- raw_spin_unlock_irqrestore.
- return event.

REQ-16: ring_buffer_consume(buffer, cpu, ts, lost_events):
- preempt_disable.
- if !cpumask_test_cpu(cpu, buffer.cpumask): event = NULL; goto out.
- cpu_buffer = buffer.buffers[cpu]; raw_spin_lock_irqsave(&reader_lock).
- event = rb_buffer_peek(cpu_buffer, ts, lost_events).
- if event ∧ event.type_len != RINGBUF_TYPE_PADDING: cpu_buffer.read++; rb_advance_reader(cpu_buffer).
- raw_spin_unlock_irqrestore; preempt_enable.
- return event.

REQ-17: ring_buffer_read_start(buffer, cpu, gfp):
- iter = kzalloc(sizeof(*iter), gfp).
- iter.event = kzalloc(BUF_MAX_DATA_SIZE).
- iter.cpu_buffer = buffer.buffers[cpu].
- atomic_inc(&cpu_buffer.resize_disabled).
- ring_buffer_iter_reset(iter)  /* iter.head_page = reader_page; cache_reader_page = reader_page; cache_pages_removed = pages_removed */
- return iter.

REQ-18: ring_buffer_resize(buffer, size, cpu_id):
- nr_pages = DIV_ROUND_UP(size, subbuf_size); min 2.
- guard(cpus_read_lock); mutex_lock(&buffer.mutex); atomic_inc(&buffer.resizing).
- if cpu_id == RING_BUFFER_ALL_CPUS:
  - For each cpu: if resize_disabled: -EBUSY.
  - For each cpu: nr_pages_to_update = nr_pages - cur; if > 0: __rb_allocate_pages into new_pages.
  - For each cpu (if pages-to-update): schedule_work_on(cpu, &update_pages_work) or run directly.
  - wait_for_completion(&update_done) per CPU.
- else: single-CPU path.
- mutex_unlock; return 0.

REQ-19: ring_buffer_swap_cpu(buffer_a, buffer_b, cpu):
- /* Both buffers must be same size, both have cpu */
- if buffer_a.subbuf_size != buffer_b.subbuf_size: -EINVAL.
- cpu_buffer_a = buffer_a.buffers[cpu]; cpu_buffer_b = buffer_b.buffers[cpu].
- atomic_inc(record_disabled both); synchronize_rcu.
- swap cpu_buffer_a / cpu_buffer_b in their parents' buffers[cpu] slots.
- swap back-pointers cpu_buffer.buffer.
- atomic_dec(record_disabled both).
- return 0.

REQ-20: trace_recursive_lock(cpu_buffer):
- bit = RB_CTX_NORMAL - interrupt_context_level()  /* 4 levels: NMI=0, IRQ=1, SOFTIRQ=2, NORMAL=3 */
- if cpu_buffer.current_context & (1 << (bit + nest)): use RB_CTX_TRANSITION fallback; if also set, RECURSION-detected → return true.
- cpu_buffer.current_context |= 1 << (bit + nest); return false.

REQ-21: rb_wakeups(buffer, cpu_buffer):
- if buffer.irq_work.waiters_pending: rb_irq_work_queue(&buffer.irq_work).
- if cpu_buffer.irq_work.waiters_pending: rb_irq_work_queue(&cpu_buffer.irq_work).
- if pages_touched changed ∧ full_waiters_pending ∧ full_hit(): wakeup_full = true; queue.

REQ-22: ring_buffer_subbuf_order_set(buffer, order):
- /* Per-resize sub-buffer order at runtime */
- mutex_lock(&buffer.mutex); for each cpu: free old pages, allocate new pages of order; subbuf_size = PAGE_SIZE << order - BUF_PAGE_HDR_SIZE; max_data_size = subbuf_size - sizeof(ring_buffer_event); rb_check_pages; mutex_unlock.

REQ-23: rb_head_page_replace(old, new):
- val = (u64)&new.list | RB_PAGE_HEAD.
- ret = cmpxchg(&old.list.prev.next, &old.list | RB_PAGE_HEAD, val).
- return ret == &old.list | RB_PAGE_HEAD.

REQ-24: Lost-events accounting:
- Per-overrun: writer overrunning head reads next_page's entries into cpu_buffer.overrun.
- Per-reader-page-swap: reader records overwrite delta into lost_events (returned via ring_buffer_consume's *lost_events out).
- Per-dropped-events: non-overwrite mode + full-buffer increments dropped_events (returned via ring_buffer_dropped_events_cpu).

REQ-25: NMI safety:
- All write paths use local_add_return / cmpxchg only (no spinlock).
- Reader paths use arch_spinlock + irq_save (not callable from NMI).
- in_nmi ∧ !ARCH_HAVE_NMI_SAFE_CMPXCHG ⟹ refuse reservation.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cmpxchg_head_flag_progress` | INVARIANT | per-rb_head_page_set: cmpxchg succeeds ⟺ pointer matched; otherwise MOVED. |
| `write_counter_monotone` | INVARIANT | per-__reserve_next: local_add_return on tail_page.write strictly advances. |
| `tail_never_passes_commit` | INVARIANT | per-move_tail: next_page == commit_page ⟹ out_reset; never overrun commit. |
| `nmi_unsafe_cmpxchg_refused` | INVARIANT | per-reserve_next: in_nmi ∧ !ARCH_NMI_SAFE_CMPXCHG ⟹ NULL. |
| `recursive_lock_excludes_same_ctx` | INVARIANT | per-trace_recursive_lock: same-ctx-bit set ⟹ TRANSITION-fallback then RECURSION. |
| `reader_page_swap_disjoint` | INVARIANT | per-get_reader_page: rb_head_page_replace cmpxchg succeeds ⟹ reader_page now where head was. |
| `overrun_accounted_on_head_takeover` | INVARIANT | per-handle_head_page: RB_PAGE_HEAD branch ⟹ overrun += entries. |
| `dropped_events_in_non_overwrite` | INVARIANT | per-move_tail: !RB_FL_OVERWRITE ∧ head-hit ⟹ dropped_events++. |
| `lost_events_delta` | INVARIANT | per-get_reader_page: lost_events = overrun - last_overrun on swap. |
| `preempt_disabled_through_commit` | INVARIANT | per-lock_reserve..unlock_commit: preempt_count > 0. |

### Layer 2: TLA+

`kernel/trace/ring-buffer.tla`:
- Per-CPU-N producers + per-1-reader + per-page-circular.
- States: { (page, write_idx, flag) } and { reader_page, head_page, tail_page, commit_page }.
- Per-producer atom: read tail_page; local_add_return write; if cross page: cmpxchg head-flag transitions HEAD→UPDATE→NORMAL.
- Properties:
  - `safety_one_head_at_a_time` — per-step: exactly one page is RB_PAGE_HEAD-marked from prev's next.
  - `safety_no_double_consume` — per-step: an event consumed by reader was not previously consumed.
  - `safety_writer_never_passes_reader_page` — per-step: tail_page swap with reader_page only via rb_head_page_replace.
  - `safety_overrun_or_drop_on_full` — per-buffer-full: either overwrite (overrun++) or drop (dropped_events++).
  - `safety_cmpxchg_linearization` — per-step: head_page transitions form a total order.
  - `liveness_reader_drains_eventually` — per-fairness: if writers stop, reader eventually returns NULL.
  - `liveness_resize_completes` — per-resize: update_pages_work runs and update_done fires.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `RingBuffer::lock_reserve` post: ret != NULL ⟹ preempt_disabled ∧ recursive-bit set | `RingBuffer::lock_reserve` |
| `RingBufferPerCpu::__reserve_next` post: ret != NULL ⟹ tail_page.write advanced by info.length | `RingBufferPerCpu::__reserve_next` |
| `RingBufferPerCpu::move_tail` post: out_reset ⟹ ret == NULL; out_again ⟹ Err(-EAGAIN) | `RingBufferPerCpu::move_tail` |
| `RingBufferPerCpu::handle_head_page` post: ret==0 ⟹ new_head set and old normalized | `RingBufferPerCpu::handle_head_page` |
| `BufferPage::head_set` post: returns ∈ {NORMAL, HEAD, UPDATE, MOVED} | `BufferPage::head_set` |
| `RingBufferPerCpu::get_reader_page` post: ret==reader_page ∧ reader_page is former head | `RingBufferPerCpu::get_reader_page` |
| `RingBuffer::consume` post: event != NULL ⟹ read advanced by event size | `RingBuffer::consume` |
| `RingBuffer::swap_cpu` post: self.buffers[cpu] / other.buffers[cpu] swapped | `RingBuffer::swap_cpu` |
| `RingBuffer::resize` post: each cpu_buffer.nr_pages == requested | `RingBuffer::resize` |

### Layer 4: Verus/Creusot functional

`Per-reserve (lock_reserve → __reserve_next → [optional rb_move_tail → handle_head_page or tail_page_update] → unlock_commit → rb_wakeups)` ≡ upstream `__rb_reserve_next` + `rb_move_tail` semantics in Documentation/trace/ring-buffer-design.rst.

`Per-read (peek → __rb_get_reader_page → rb_head_page_replace → rb_advance_reader)` ≡ upstream `rb_buffer_peek`.

`Per-resize (ring_buffer_resize → __rb_allocate_pages → update_pages_handler)` ≡ atomic add/remove of buffer_page nodes preserving event order modulo concurrent commits.

### hardening

(Inherits row-1 features from `kernel/trace/00-overview.md` § Hardening.)

Ring-buffer reinforcement:

- **Per-cmpxchg head-flag protocol** — defense against per-multi-writer torn pointer update.
- **Per-arch_spinlock internal head-swap** — defense against per-reader/writer concurrent head-replace.
- **Per-trace_recursive_lock 5-bit context mask** — defense against per-self-recursion deadlock.
- **Per-in_nmi cmpxchg-safety gate** — defense against per-NMI deadlock on non-NMI-safe-cmpxchg.
- **Per-RB_WARN_ON bounded retries (3 loops)** — defense against per-livelock on head-page swap.
- **Per-1-second USECS_WAIT writer-completion timeout** — defense against per-stuck-writer reader-hang.
- **Per-record_disabled atomic counter** — defense against per-disable-race writer-after-disable.
- **Per-resize_disabled per-iter atomic** — defense against per-resize-during-iteration corruption.
- **Per-cpus_read_lock around resize** — defense against per-CPU-hotplug-race resize.
- **Per-irq_work for reader wake** — defense against per-context-unsafe direct wake.
- **Per-local_t per-CPU counters (no atomic_t)** — defense against per-cache-line-bounce hot-path slowdown.
- **Per-RB_FL_OVERWRITE vs drop dichotomy** — defense against per-silent-loss in non-overwrite mode.
- **Per-lost_events propagation to userspace** — defense against per-silent-overrun blind spot.
- **Per-MAX_NEST recursion bound on event_stamp** — defense against per-overflow event_stamp[].
- **Per-rb_check_pages after disable** — defense against per-corrupted-list undetected.
- **Per-`__GFP_RETRY_MAYFAIL | __GFP_COMP | __GFP_ZERO` for data pages** — defense against per-uninitialized-leak and per-OOM-cascade.

