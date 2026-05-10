# Tier-3: kernel/bpf/ringbuf.c — BPF ring buffer map

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/bpf/00-overview.md
upstream-paths:
  - kernel/bpf/ringbuf.c (~880 lines)
  - include/uapi/linux/bpf.h (BPF_MAP_TYPE_RINGBUF, BPF_MAP_TYPE_USER_RINGBUF, BPF_RB_*, BPF_RINGBUF_*)
  - include/linux/bpf.h (struct bpf_map, struct bpf_map_ops)
-->

## Summary

The **BPF ring buffer** is a many-producer, single-consumer (MPSC) lockless-ish message channel between kernel BPF programs and a user-space reader. Per-map `struct bpf_ringbuf_map` wraps a `struct bpf_ringbuf` which contains: per-data area (power-of-two pages, double-mapped so wrap-around reads are linear), per-producer page (`producer_pos`), per-consumer page (`consumer_pos`), per-non-mmappable header (waitq, spinlock, mask, pending_pos, overwrite_pos). Per-record carries an 8-byte header `struct bpf_ringbuf_hdr { u32 len; u32 pg_off; }` where `len` has `BPF_RINGBUF_BUSY_BIT` (in-progress) and `BPF_RINGBUF_DISCARD_BIT` (drop) flags. Per-BPF helper: `bpf_ringbuf_reserve(map, size, flags)` (reserve), `bpf_ringbuf_submit(rec, flags)` / `bpf_ringbuf_discard(rec, flags)` (commit/abort), `bpf_ringbuf_output(map, data, size, flags)` (atomic reserve+memcpy+submit), `bpf_ringbuf_query(map, flag)` (counters). Per-wakeup flags: `BPF_RB_NO_WAKEUP` (suppress), `BPF_RB_FORCE_WAKEUP` (always notify). Per-`BPF_MAP_TYPE_USER_RINGBUF` reverses producer/consumer roles: user-space writes, kernel drains via `bpf_user_ringbuf_drain(map, cb, ctx, flags)`. Per-poll: `EPOLLIN` (kernel-producer) or `EPOLLOUT` (user-producer). Critical for: tracing data export, perf-event replacement, sk_msg redirection, low-overhead kernel→user telemetry.

This Tier-3 covers `kernel/bpf/ringbuf.c` (~880 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bpf_ringbuf` | per-instance state | `BpfRingbuf` |
| `struct bpf_ringbuf_map` | per-map wrapper | `BpfRingbufMap` |
| `struct bpf_ringbuf_hdr` | per-record 8-byte header | `BpfRingbufHdr` |
| `bpf_ringbuf_area_alloc()` | per-alloc pages + double-map vmap | `BpfRingbuf::area_alloc` |
| `bpf_ringbuf_alloc()` | per-init (spinlock, waitq, irq_work) | `BpfRingbuf::alloc` |
| `bpf_ringbuf_free()` | per-free (vunmap + free_pages) | `BpfRingbuf::free` |
| `ringbuf_map_alloc()` | `.map_alloc` per-attr validation | `BpfRingbufMap::map_alloc` |
| `ringbuf_map_free()` | `.map_free` | `BpfRingbufMap::map_free` |
| `ringbuf_map_mmap_kern()` | kernel-producer mmap policy | `BpfRingbufMap::mmap_kern` |
| `ringbuf_map_mmap_user()` | user-producer mmap policy | `BpfRingbufMap::mmap_user` |
| `ringbuf_map_poll_kern()` | per-EPOLLIN poll | `BpfRingbufMap::poll_kern` |
| `ringbuf_map_poll_user()` | per-EPOLLOUT poll | `BpfRingbufMap::poll_user` |
| `__bpf_ringbuf_reserve()` | per-reserve (kernel producer) | `BpfRingbuf::reserve` |
| `bpf_ringbuf_commit()` | per-submit/discard finalize | `BpfRingbuf::commit` |
| `bpf_ringbuf_reserve` helper | BPF helper id `BPF_FUNC_ringbuf_reserve` | `BpfRingbuf::helper_reserve` |
| `bpf_ringbuf_submit` helper | BPF helper id `BPF_FUNC_ringbuf_submit` | `BpfRingbuf::helper_submit` |
| `bpf_ringbuf_discard` helper | BPF helper id `BPF_FUNC_ringbuf_discard` | `BpfRingbuf::helper_discard` |
| `bpf_ringbuf_output` helper | per-atomic reserve+copy+submit | `BpfRingbuf::helper_output` |
| `bpf_ringbuf_query` helper | per-counter readout | `BpfRingbuf::helper_query` |
| `bpf_ringbuf_reserve_dynptr` helper | per-dynptr reserve | `BpfRingbuf::helper_reserve_dynptr` |
| `bpf_ringbuf_submit_dynptr` helper | per-dynptr submit | `BpfRingbuf::helper_submit_dynptr` |
| `bpf_ringbuf_discard_dynptr` helper | per-dynptr discard | `BpfRingbuf::helper_discard_dynptr` |
| `__bpf_user_ringbuf_peek()` | per-user-producer sample peek | `BpfRingbuf::user_peek` |
| `__bpf_user_ringbuf_sample_release()` | per-user-producer commit | `BpfRingbuf::user_release` |
| `bpf_user_ringbuf_drain` helper | per-drain (consume loop) | `BpfRingbuf::helper_user_drain` |
| `bpf_ringbuf_notify()` | per-irq_work waitq wakeup | `BpfRingbuf::notify` |
| `bpf_ringbuf_has_space()` | per-reserve space check | `BpfRingbuf::has_space` |
| `bpf_ringbuf_rec_pg_off()` | per-record→rb page offset | `BpfRingbuf::rec_pg_off` |
| `bpf_ringbuf_restore_from_rec()` | per-record→rb pointer restore | `BpfRingbuf::restore_from_rec` |
| `ringbuf_avail_data_sz()` | per-poll estimate | `BpfRingbuf::avail_data_sz` |
| `ringbuf_map_ops` / `user_ringbuf_map_ops` | per-map_ops vtable | `BPF_RINGBUF_MAP_OPS` / `BPF_USER_RINGBUF_MAP_OPS` |

## Compatibility contract

REQ-1: struct bpf_ringbuf:
- waitq: wait_queue_head_t for `poll()` blockers.
- work: struct irq_work for deferred wakeup from BPF context.
- mask: data_sz - 1 (data area is power-of-two).
- pages: struct page ** (raw page array; double-mapped).
- nr_pages: count of (meta + data) pages (NOT including the second mapping of data).
- overwrite_mode: bool (BPF_F_RB_OVERWRITE).
- spinlock: rqspinlock_t (recursion-safe) — per-reserve serialization.
- busy: atomic_t — per-user-ringbuf concurrent-consumer guard.
- consumer_pos: unsigned long (page-aligned; on its own page; writable by user in kernel-producer mode).
- producer_pos: unsigned long (page-aligned; on its own page; r/o to user in kernel-producer mode).
- pending_pos: unsigned long (oldest in-flight reservation).
- overwrite_pos: unsigned long (BPF_F_RB_OVERWRITE only: position past last overwritten record).
- data[]: PAGE_SIZE-aligned ring data (mapped twice contiguously in virtual space).

REQ-2: struct bpf_ringbuf_map { struct bpf_map map; struct bpf_ringbuf *rb; }.

REQ-3: struct bpf_ringbuf_hdr { u32 len; u32 pg_off; }:
- len: payload size in bytes; high bits = BPF_RINGBUF_BUSY_BIT (1<<31), BPF_RINGBUF_DISCARD_BIT (1<<30).
- pg_off: header page offset from start of rb (used to restore rb pointer from record pointer).
- Total header size = BPF_RINGBUF_HDR_SZ = 8 bytes; records padded to multiple of 8.

REQ-4: ringbuf_map_alloc(attr):
- /* Validate flags */
- if attr.map_flags & ~(BPF_F_NUMA_NODE | BPF_F_RB_OVERWRITE): return -EINVAL.
- /* OVERWRITE only with RINGBUF, not USER_RINGBUF */
- if attr.map_flags & BPF_F_RB_OVERWRITE ∧ attr.map_type != BPF_MAP_TYPE_RINGBUF: return -EINVAL.
- /* Key/value: ringbuf has no key/value; max_entries = data-area bytes */
- if attr.key_size != 0 ∨ attr.value_size != 0: return -EINVAL.
- if !is_power_of_2(attr.max_entries) ∨ !PAGE_ALIGNED(attr.max_entries): return -EINVAL.
- /* Allocate rb_map + rb */
- rb_map = bpf_map_area_alloc(sizeof(*rb_map), NUMA_NO_NODE).
- bpf_map_init_from_attr(&rb_map.map, attr).
- rb_map.rb = bpf_ringbuf_alloc(attr.max_entries, rb_map.map.numa_node, overwrite_mode).
- return &rb_map.map.

REQ-5: bpf_ringbuf_area_alloc(data_sz, numa_node):
- nr_meta_pages = RINGBUF_PGOFF + 2 = (offsetof(consumer_pos) >> PAGE_SHIFT) + 2.
- nr_data_pages = data_sz >> PAGE_SHIFT.
- nr_pages = nr_meta_pages + nr_data_pages.
- /* Allocate page array sized for double-mapping of data */
- array_size = (nr_meta_pages + 2 * nr_data_pages) * sizeof(struct page *).
- pages = bpf_map_area_alloc(array_size, numa_node).
- for i in 0..nr_pages:
  - page = alloc_pages_node(numa_node, GFP_KERNEL_ACCOUNT | __GFP_RETRY_MAYFAIL | __GFP_NOWARN | __GFP_ZERO, 0).
  - pages[i] = page.
  - if i >= nr_meta_pages: pages[nr_data_pages + i] = page. /* mirror */
- rb = vmap(pages, nr_meta_pages + 2 * nr_data_pages, VM_MAP | VM_USERMAP, PAGE_KERNEL).
- kmemleak_not_leak(pages); rb.pages = pages; rb.nr_pages = nr_pages.

REQ-6: bpf_ringbuf_alloc(data_sz, numa_node, overwrite_mode):
- rb = area_alloc(data_sz, numa_node).
- raw_res_spin_lock_init(&rb.spinlock).
- atomic_set(&rb.busy, 0).
- init_waitqueue_head(&rb.waitq).
- init_irq_work(&rb.work, bpf_ringbuf_notify).
- rb.mask = data_sz - 1.
- rb.consumer_pos = 0; rb.producer_pos = 0; rb.pending_pos = 0.
- rb.overwrite_mode = overwrite_mode.

REQ-7: ringbuf_map_mmap_kern(map, vma) — kernel-producer mmap:
- /* Writable only for consumer_pos page (vm_pgoff == 0, size == PAGE_SIZE) */
- if vma.vm_flags & VM_WRITE:
  - if vma.vm_pgoff != 0 ∨ vma_size != PAGE_SIZE: return -EPERM.
- return remap_vmalloc_range(vma, rb, vma.vm_pgoff + RINGBUF_PGOFF).

REQ-8: ringbuf_map_mmap_user(map, vma) — user-producer mmap:
- /* Producer + data writable; consumer page r/o to user */
- if vma.vm_flags & VM_WRITE:
  - if vma.vm_pgoff == 0: return -EPERM. /* consumer page */
- return remap_vmalloc_range(vma, rb, vma.vm_pgoff + RINGBUF_PGOFF).

REQ-9: __bpf_ringbuf_reserve(rb, size) — kernel-producer reserve:
- if size > RINGBUF_MAX_RECORD_SZ (UINT_MAX/4): return NULL.
- len = round_up(size + BPF_RINGBUF_HDR_SZ, 8).
- if len > ringbuf_total_data_sz(rb): return NULL.
- cons_pos = smp_load_acquire(&rb.consumer_pos).
- if raw_res_spin_lock_irqsave(&rb.spinlock, flags): return NULL.
- pend_pos = rb.pending_pos; prod_pos = rb.producer_pos.
- new_prod_pos = prod_pos + len.
- /* Advance pending_pos past committed records */
- while pend_pos < prod_pos:
  - hdr = rb.data + (pend_pos & rb.mask).
  - hdr_len = READ_ONCE(hdr.len).
  - if hdr_len & BPF_RINGBUF_BUSY_BIT: break.
  - pend_pos += round_up((hdr_len & ~BPF_RINGBUF_DISCARD_BIT) + BPF_RINGBUF_HDR_SZ, 8).
- rb.pending_pos = pend_pos.
- if !bpf_ringbuf_has_space(rb, new_prod_pos, cons_pos, pend_pos):
  - unlock; return NULL.
- /* Overwrite mode: advance overwrite_pos past records being discarded */
- if rb.overwrite_mode:
  - over_pos = rb.overwrite_pos.
  - while new_prod_pos - over_pos > rb.mask:
    - hdr = rb.data + (over_pos & rb.mask).
    - over_pos += round_up((READ_ONCE(hdr.len) & ~BPF_RINGBUF_DISCARD_BIT) + BPF_RINGBUF_HDR_SZ, 8).
  - WRITE_ONCE(rb.overwrite_pos, over_pos).
- /* Write header */
- hdr = rb.data + (prod_pos & rb.mask).
- hdr.len = size | BPF_RINGBUF_BUSY_BIT.
- hdr.pg_off = bpf_ringbuf_rec_pg_off(rb, hdr).
- /* Release-publish new producer position */
- smp_store_release(&rb.producer_pos, new_prod_pos).
- unlock.
- return hdr + BPF_RINGBUF_HDR_SZ.

REQ-10: bpf_ringbuf_has_space(rb, new_prod_pos, cons_pos, pend_pos):
- /* Oldest in-flight cannot fall more than ringbuf_size-1 behind newest */
- if new_prod_pos - pend_pos > rb.mask: return false.
- if rb.overwrite_mode: return true.
- /* Producer cannot lap consumer */
- if new_prod_pos - cons_pos > rb.mask: return false.
- return true.

REQ-11: bpf_ringbuf_commit(sample, flags, discard):
- hdr = sample - BPF_RINGBUF_HDR_SZ.
- rb = bpf_ringbuf_restore_from_rec(hdr). /* uses hdr.pg_off + PAGE_MASK */
- new_len = hdr.len ^ BPF_RINGBUF_BUSY_BIT. /* clear busy */
- if discard: new_len |= BPF_RINGBUF_DISCARD_BIT.
- xchg(&hdr.len, new_len). /* atomic publish */
- /* Wakeup decision */
- rec_pos = (hdr - rb.data).
- cons_pos = smp_load_acquire(&rb.consumer_pos) & rb.mask.
- if flags & BPF_RB_FORCE_WAKEUP: irq_work_queue(&rb.work).
- else if cons_pos == rec_pos ∧ !(flags & BPF_RB_NO_WAKEUP): irq_work_queue(&rb.work).

REQ-12: bpf_ringbuf_reserve / _submit / _discard helpers (kernel-producer):
- bpf_ringbuf_reserve(map, size, flags): if flags != 0 → 0. else return __bpf_ringbuf_reserve(rb, size).
- bpf_ringbuf_submit(sample, flags): commit(sample, flags, false). returns void.
- bpf_ringbuf_discard(sample, flags): commit(sample, flags, true). returns void.
- /* Verifier-enforced: reserve returns RET_PTR_TO_RINGBUF_MEM_OR_NULL, submit/discard take OBJ_RELEASE */

REQ-13: bpf_ringbuf_output(map, data, size, flags):
- /* Atomic reserve + memcpy + submit */
- if flags & ~(BPF_RB_NO_WAKEUP | BPF_RB_FORCE_WAKEUP): return -EINVAL.
- rec = __bpf_ringbuf_reserve(rb, size).
- if !rec: return -EAGAIN.
- memcpy(rec, data, size).
- bpf_ringbuf_commit(rec, flags, false).
- return 0.

REQ-14: bpf_ringbuf_query(map, flag):
- BPF_RB_AVAIL_DATA → ringbuf_avail_data_sz(rb).
- BPF_RB_RING_SIZE → rb.mask + 1.
- BPF_RB_CONS_POS → smp_load_acquire(&rb.consumer_pos).
- BPF_RB_PROD_POS → smp_load_acquire(&rb.producer_pos).
- BPF_RB_OVERWRITE_POS → smp_load_acquire(&rb.overwrite_pos).
- default → 0.

REQ-15: ringbuf_avail_data_sz(rb):
- cons_pos = smp_load_acquire(&rb.consumer_pos).
- if rb.overwrite_mode:
  - over_pos = smp_load_acquire(&rb.overwrite_pos).
  - prod_pos = smp_load_acquire(&rb.producer_pos).
  - return prod_pos - max(cons_pos, over_pos).
- else:
  - prod_pos = smp_load_acquire(&rb.producer_pos).
  - return prod_pos - cons_pos.
- Note: estimate; not synchronized with producers.

REQ-16: poll integration:
- ringbuf_map_poll_kern: poll_wait(filp, &rb.waitq, pts); if avail_data_sz(rb) > 0 → EPOLLIN | EPOLLRDNORM.
- ringbuf_map_poll_user: poll_wait(filp, &rb.waitq, pts); if avail < total_data_sz → EPOLLOUT | EPOLLWRNORM.
- bpf_ringbuf_notify is the irq_work handler that calls wake_up_all(&rb.waitq).

REQ-17: user-ringbuf consumer side (`__bpf_user_ringbuf_peek` / `_release` / `bpf_user_ringbuf_drain`):
- /* Producer = user-space; consumer = kernel BPF program */
- bpf_user_ringbuf_drain(map, callback_fn, callback_ctx, flags):
  - if flags & ~(BPF_RB_NO_WAKEUP | BPF_RB_FORCE_WAKEUP): return -EINVAL.
  - /* Mutual exclusion via atomic busy bit (NOT spinlock — callback may sleep relative to IRQ ctx) */
  - if !atomic_try_cmpxchg(&rb.busy, &0, 1): return -EBUSY.
  - for samples in 0..BPF_MAX_USER_RINGBUF_SAMPLES while ret == 0:
    - err = __bpf_user_ringbuf_peek(rb, &sample, &size).
    - if err == -ENODATA: break (queue empty).
    - if err == -EAGAIN: discarded++; continue (skipped DISCARD bit).
    - if err: ret = err; goto release.
    - bpf_dynptr_init(&dynptr, sample, BPF_DYNPTR_TYPE_LOCAL, 0, size).
    - ret = callback(&dynptr, callback_ctx, 0, 0, 0).
    - __bpf_user_ringbuf_sample_release(rb, size, flags).
  - ret = samples - discarded_samples.
  - release: atomic_set_release(&rb.busy, 0).
  - /* Wake user-space producers waiting on EPOLLOUT */
  - if flags & BPF_RB_FORCE_WAKEUP: irq_work_queue.
  - else if !(flags & BPF_RB_NO_WAKEUP) ∧ samples > 0: irq_work_queue.

REQ-18: __bpf_user_ringbuf_peek validation (paranoid — user-supplied):
- prod_pos = smp_load_acquire(&rb.producer_pos).
- if prod_pos % 8 != 0: return -EINVAL.
- cons_pos = smp_load_acquire(&rb.consumer_pos).
- if cons_pos >= prod_pos: return -ENODATA.
- hdr = rb.data + (cons_pos & rb.mask).
- hdr_len = smp_load_acquire(hdr).
- flags = hdr_len & (BUSY|DISCARD); sample_len = hdr_len & ~flags.
- total_len = round_up(sample_len + BPF_RINGBUF_HDR_SZ, 8).
- if total_len > prod_pos - cons_pos: return -EINVAL.
- if total_len > ringbuf_total_data_sz(rb): return -E2BIG.
- if bpf_dynptr_check_size(sample_len) != 0: return -E2BIG.
- if flags & BPF_RINGBUF_DISCARD_BIT:
  - smp_store_release(&rb.consumer_pos, cons_pos + total_len). /* advance past skip */
  - return -EAGAIN.
- if flags & BPF_RINGBUF_BUSY_BIT: return -ENODATA.
- *sample = rb.data + ((cons_pos + BPF_RINGBUF_HDR_SZ) & rb.mask); *size = sample_len.

## Acceptance Criteria

- [ ] AC-1: `map_alloc` rejects non-power-of-two / non-page-aligned `max_entries` with -EINVAL.
- [ ] AC-2: `map_alloc` rejects BPF_F_RB_OVERWRITE on USER_RINGBUF.
- [ ] AC-3: `mmap_kern` denies writable mapping outside the consumer_pos page.
- [ ] AC-4: `mmap_user` denies writable mapping on the consumer_pos page.
- [ ] AC-5: Concurrent producers serialize via `rb.spinlock`; reserve returns NULL when full and not in overwrite mode.
- [ ] AC-6: BPF_F_RB_OVERWRITE: reserve never returns NULL for fit-sized records; overwrite_pos advances past dropped records.
- [ ] AC-7: `bpf_ringbuf_submit` clears BUSY bit and publishes record via atomic xchg.
- [ ] AC-8: `bpf_ringbuf_discard` sets DISCARD bit and consumer skips the slot.
- [ ] AC-9: BPF_RB_FORCE_WAKEUP queues irq_work unconditionally; BPF_RB_NO_WAKEUP suppresses even when consumer caught up.
- [ ] AC-10: poll: EPOLLIN seen iff `avail_data_sz > 0` for kernel-producer; EPOLLOUT iff space free for user-producer.
- [ ] AC-11: `bpf_ringbuf_query(BPF_RB_*_POS)` returns acquire-loaded counters; AVAIL_DATA matches producer−max(consumer,overwrite).
- [ ] AC-12: `bpf_user_ringbuf_drain` enforces `BPF_MAX_USER_RINGBUF_SAMPLES` upper bound per call.
- [ ] AC-13: `bpf_user_ringbuf_drain` returns -EBUSY if another consumer holds the busy bit.
- [ ] AC-14: user-ringbuf peek rejects unaligned producer_pos, oversized samples, or sample crossing producer_pos boundary.
- [ ] AC-15: Free path: irq_work_sync, vunmap, __free_page for each, bpf_map_area_free for page array.

## Architecture

```
struct BpfRingbuf {
  waitq:           WaitQueueHead,
  work:            IrqWork,
  mask:            u64,                   // data_sz - 1
  pages:           *mut *mut Page,
  nr_pages:        u32,
  overwrite_mode:  bool,
  spinlock:        RqSpinLock,            // cacheline-aligned
  busy:            AtomicU32,             // cacheline-aligned; user-ringbuf consumer guard
  consumer_pos:    UnsafeCell<u64>,       // PAGE_SIZE-aligned
  producer_pos:    UnsafeCell<u64>,       // PAGE_SIZE-aligned
  pending_pos:     u64,                   // protected by spinlock
  overwrite_pos:   UnsafeCell<u64>,       // overwrite-mode only
  data:            [u8],                  // VLA, PAGE_SIZE-aligned, double-vmap'd
}

struct BpfRingbufMap {
  map: BpfMap,
  rb:  *mut BpfRingbuf,
}

#[repr(C)]
struct BpfRingbufHdr {
  len:    u32,      // payload | BUSY_BIT | DISCARD_BIT
  pg_off: u32,      // hdr page offset from start of rb
}
```

`BpfRingbuf::area_alloc(data_sz, numa_node) -> Option<*mut BpfRingbuf>`:
1. nr_meta_pages = RINGBUF_NR_META_PAGES = (offsetof(consumer_pos) >> PAGE_SHIFT) + RINGBUF_POS_PAGES (= +2).
2. nr_data_pages = data_sz / PAGE_SIZE.
3. nr_pages = nr_meta_pages + nr_data_pages.
4. /* Allocate page array sized for one meta-set + two data-set mirrors */
5. array_size = (nr_meta_pages + 2 * nr_data_pages) * sizeof::<*mut Page>().
6. pages = bpf_map_area_alloc(array_size, numa_node).
7. for i in 0..nr_pages:
   - page = alloc_pages_node(numa_node, GFP_KERNEL_ACCOUNT | RETRY_MAYFAIL | NOWARN | ZERO, 0).
   - pages[i] = page.
   - if i >= nr_meta_pages: pages[nr_data_pages + i] = page.   /* mirror for wrap */
8. rb = vmap(pages, nr_meta_pages + 2*nr_data_pages, VM_MAP | VM_USERMAP, PAGE_KERNEL).
9. rb.pages = pages; rb.nr_pages = nr_pages.

`BpfRingbuf::reserve(rb, size) -> Option<*mut u8>`:
1. if size > RINGBUF_MAX_RECORD_SZ (= UINT_MAX/4): return None.
2. len = round_up(size + BPF_RINGBUF_HDR_SZ, 8).
3. if len > rb.mask + 1: return None.
4. cons_pos = smp_load_acquire(&rb.consumer_pos).
5. /* rqspinlock returns non-zero on cycle/timeout */
6. if raw_res_spin_lock_irqsave(&rb.spinlock, &flags) != 0: return None.
7. pend_pos = rb.pending_pos; prod_pos = rb.producer_pos.
8. new_prod_pos = prod_pos + len.
9. /* Advance pending past contiguous-committed prefix */
10. while pend_pos < prod_pos:
    - hdr = rb.data + (pend_pos & rb.mask).
    - hdr_len = READ_ONCE(hdr.len).
    - if hdr_len & BUSY_BIT: break.
    - pend_pos += round_up((hdr_len & ~DISCARD_BIT) + BPF_RINGBUF_HDR_SZ, 8).
11. rb.pending_pos = pend_pos.
12. if !Self::has_space(rb, new_prod_pos, cons_pos, pend_pos):
    - unlock; return None.
13. if rb.overwrite_mode:
    - over_pos = rb.overwrite_pos.
    - while new_prod_pos - over_pos > rb.mask:
      - hdr = rb.data + (over_pos & rb.mask).
      - over_pos += round_up((READ_ONCE(hdr.len) & ~DISCARD_BIT) + BPF_RINGBUF_HDR_SZ, 8).
    - WRITE_ONCE(rb.overwrite_pos, over_pos).
14. hdr = rb.data + (prod_pos & rb.mask).
15. hdr.len = (size as u32) | BUSY_BIT.
16. hdr.pg_off = ((hdr - rb) >> PAGE_SHIFT) as u32.
17. smp_store_release(&rb.producer_pos, new_prod_pos).
18. unlock.
19. return Some(hdr + BPF_RINGBUF_HDR_SZ).

`BpfRingbuf::commit(sample, flags, discard)`:
1. hdr = sample - BPF_RINGBUF_HDR_SZ.
2. rb = Self::restore_from_rec(hdr). /* (hdr & PAGE_MASK) - (hdr.pg_off << PAGE_SHIFT) */
3. new_len = hdr.len ^ BUSY_BIT.
4. if discard: new_len |= DISCARD_BIT.
5. xchg(&hdr.len, new_len). /* atomic publish */
6. rec_pos = hdr - rb.data.
7. cons_pos = smp_load_acquire(&rb.consumer_pos) & rb.mask.
8. if flags & BPF_RB_FORCE_WAKEUP: irq_work_queue(&rb.work).
9. else if cons_pos == rec_pos ∧ !(flags & BPF_RB_NO_WAKEUP): irq_work_queue(&rb.work).

`BpfRingbuf::user_peek(rb) -> Result<(*mut u8, u32), Err>`:
1. prod_pos = smp_load_acquire(&rb.producer_pos).
2. if prod_pos % 8 != 0: return Err(-EINVAL). /* user tampered */
3. cons_pos = smp_load_acquire(&rb.consumer_pos).
4. if cons_pos >= prod_pos: return Err(-ENODATA).
5. hdr_ptr = rb.data + (cons_pos & rb.mask) as *const u32.
6. hdr_len = smp_load_acquire(hdr_ptr).
7. f = hdr_len & (BUSY_BIT|DISCARD_BIT).
8. sample_len = hdr_len & !f.
9. total_len = round_up(sample_len + BPF_RINGBUF_HDR_SZ, 8).
10. if total_len > (prod_pos - cons_pos): return Err(-EINVAL).
11. if total_len > rb.mask + 1: return Err(-E2BIG).
12. if bpf_dynptr_check_size(sample_len).is_err(): return Err(-E2BIG).
13. if f & DISCARD_BIT:
    - smp_store_release(&rb.consumer_pos, cons_pos + total_len).
    - return Err(-EAGAIN).
14. if f & BUSY_BIT: return Err(-ENODATA).
15. sample = rb.data + ((cons_pos + BPF_RINGBUF_HDR_SZ) & rb.mask).
16. Ok((sample, sample_len)).

`BpfRingbuf::helper_user_drain(map, cb, ctx, flags) -> i64`:
1. if flags & !(BPF_RB_NO_WAKEUP | BPF_RB_FORCE_WAKEUP): return -EINVAL.
2. if atomic_try_cmpxchg(&rb.busy, 0, 1).is_err(): return -EBUSY.
3. samples = 0; discarded = 0; ret = 0.
4. for samples in 0..BPF_MAX_USER_RINGBUF_SAMPLES while ret == 0:
   - match Self::user_peek(rb):
     - Err(-ENODATA): break.
     - Err(-EAGAIN): discarded += 1; continue.
     - Err(e): ret = e; goto schedule_work_return.
     - Ok((sample, size)):
       - bpf_dynptr_init(&dynptr, sample, BPF_DYNPTR_TYPE_LOCAL, 0, size).
       - ret = cb(&dynptr, ctx, 0, 0, 0).
       - Self::user_release(rb, size, flags). /* advances consumer_pos */
5. ret = samples - discarded.
6. atomic_set_release(&rb.busy, 0).
7. if flags & BPF_RB_FORCE_WAKEUP: irq_work_queue(&rb.work).
8. else if !(flags & BPF_RB_NO_WAKEUP) ∧ samples > 0: irq_work_queue(&rb.work).
9. return ret.

`BpfRingbuf::notify(work)`:
1. rb = container_of(work, BpfRingbuf, work).
2. wake_up_all(&rb.waitq).  /* satisfies poll(POLLIN/POLLOUT) waiters */

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `record_busy_bit_inverted_on_commit` | INVARIANT | per-commit: new_len = hdr.len ^ BUSY_BIT; BUSY cleared on submit/discard. |
| `producer_pos_release_paired_with_acquire` | INVARIANT | smp_store_release(producer_pos) pairs with smp_load_acquire(consumer side). |
| `pending_pos_monotonic_nondecreasing` | INVARIANT | per-reserve: rb.pending_pos never decreases. |
| `overwrite_pos_only_in_overwrite_mode` | INVARIANT | overwrite_pos written ⟹ rb.overwrite_mode == true. |
| `reserve_returns_aligned_hdr_ptr` | INVARIANT | reserve return value points at hdr+8; hdr is 8-byte aligned. |
| `consumer_pos_writable_only_by_consumer_page` | INVARIANT | per-mmap_kern: VM_WRITE allowed only when vm_pgoff==0 ∧ size==PAGE_SIZE. |
| `consumer_pos_readonly_for_user_producer` | INVARIANT | per-mmap_user: VM_WRITE denied when vm_pgoff==0. |
| `busy_bit_held_across_drain_callback` | INVARIANT | per-bpf_user_ringbuf_drain: atomic busy = 1 across callback; cleared via atomic_set_release. |
| `user_peek_rejects_unaligned_producer_pos` | INVARIANT | prod_pos % 8 != 0 ⟹ -EINVAL. |
| `user_peek_rejects_oversized_total_len` | INVARIANT | total_len > (prod_pos-cons_pos) ∨ total_len > data_sz ⟹ error. |

### Layer 2: TLA+

`kernel/bpf/ringbuf.tla`:
- Per-reserve + per-commit + per-consume + per-overwrite + per-wakeup.
- Properties:
  - `safety_no_double_publish` — per-record: BUSY_BIT cleared exactly once.
  - `safety_no_record_overlap` — per-prod_pos: each record's [prod_pos, prod_pos+len) disjoint while busy.
  - `safety_consumer_never_passes_producer` — cons_pos ≤ prod_pos always.
  - `safety_overwrite_only_when_overwrite_mode` — overwrite_pos advances ⟹ overwrite_mode.
  - `safety_busy_bit_serializes_user_consumer` — per-bpf_user_ringbuf_drain: at most one consumer at a time.
  - `safety_force_wakeup_always_notifies` — BPF_RB_FORCE_WAKEUP ⟹ irq_work_queue called.
  - `safety_no_wakeup_never_notifies` — BPF_RB_NO_WAKEUP ∧ !FORCE ⟹ no irq_work_queue.
  - `liveness_per_reserve_terminates` — per-reserve: spinlock+space-check always returns (rqspinlock breaks cycles).
  - `liveness_per_submit_eventually_visible` — per-commit: consumer eventually sees cleared BUSY_BIT.
  - `liveness_per_poller_eventually_wakes` — per-irq_work_queue ⟹ wake_up_all dispatched.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `BpfRingbuf::area_alloc` post: nr_pages*PAGE_SIZE == data_sz + nr_meta_pages*PAGE_SIZE; data pages mirrored | `BpfRingbuf::area_alloc` |
| `BpfRingbuf::reserve` post: ret == NULL ∨ ret = hdr+8 ∧ hdr.len & BUSY_BIT ∧ hdr.pg_off restores rb | `BpfRingbuf::reserve` |
| `BpfRingbuf::commit` post: hdr.len's BUSY_BIT cleared; DISCARD_BIT set iff discard | `BpfRingbuf::commit` |
| `BpfRingbuf::has_space` post: false ⟹ reserve must abort | `BpfRingbuf::has_space` |
| `BpfRingbuf::avail_data_sz` post: returns prod_pos - max(cons_pos, overwrite_pos[opt]) | `BpfRingbuf::avail_data_sz` |
| `BpfRingbuf::user_peek` post: total_len ≤ prod_pos-cons_pos ∧ ≤ data_sz | `BpfRingbuf::user_peek` |
| `BpfRingbuf::helper_user_drain` post: samples ≤ BPF_MAX_USER_RINGBUF_SAMPLES; busy=0 on return | `BpfRingbuf::helper_user_drain` |
| `BpfRingbuf::notify` post: waitq waiters dequeued | `BpfRingbuf::notify` |
| `ringbuf_map_mmap_kern` post: VM_WRITE ⟹ (vm_pgoff==0 ∧ size==PAGE_SIZE) | `BpfRingbufMap::mmap_kern` |
| `ringbuf_map_mmap_user` post: VM_WRITE ∧ vm_pgoff==0 ⟹ -EPERM | `BpfRingbufMap::mmap_user` |

### Layer 4: Verus/Creusot functional

`Per-producer reserve → write header(BUSY) → smp_store_release(producer_pos) → consumer smp_load_acquire(producer_pos) → see new prod_pos → read hdr.len → on BUSY=0 process record → smp_store_release(consumer_pos)` happens-before chain: per-Documentation/bpf/ringbuf.rst.

`Per-user-producer write → user smp_store_release(producer_pos) → kernel __bpf_user_ringbuf_peek smp_load_acquire → callback → __bpf_user_ringbuf_sample_release smp_store_release(consumer_pos) → user smp_load_acquire(consumer_pos) → free space` happens-before chain: per-libbpf user_ring_buffer__reserve documentation.

## Hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

Ringbuf-specific reinforcement:

- **Per-rqspinlock reserve** — defense against per-deadlock from re-entrant BPF (tracepoint inside locked region).
- **Per-IRQ-deferred wakeup (irq_work)** — defense against per-wake_up_all-in-NMI context.
- **Per-consumer page r/o for kernel-producer mmap** — defense against user tampering with producer counter.
- **Per-consumer page r/o for user-producer mmap, but data + producer page r/w** — clear permission split.
- **Per-double-page-mapping vmap** — defense against wrap-around copy bugs (record always contiguous virtually).
- **Per-BPF_RINGBUF_BUSY_BIT atomic-clear via xchg** — defense against partial-publication races.
- **Per-producer position release-store + consumer acquire-load** — defense against record-content reordering.
- **Per-atomic busy flag for user-ringbuf consumer** — defense against concurrent drain corrupting consumer_pos.
- **Per-paranoid user-supplied header validation** — defense against malicious user-producer (length > prod-cons, length > data_sz, unaligned prod_pos).
- **Per-DISCARD bit advances consumer past poisoned slot** — defense against stuck-consumer DoS.
- **Per-RINGBUF_MAX_RECORD_SZ = UINT_MAX/4** — defense against integer overflow in len computation.
- **Per-BPF_MAX_USER_RINGBUF_SAMPLES cap on drain loop** — defense against per-callback infinite drain (RCU stall, scheduler starvation).
- **Per-irq_work_sync on free** — defense against per-UAF from in-flight wakeup at teardown.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/bpf/syscall.c BPF_MAP_CREATE plumbing (covered in `bpf-core.md` Tier-3)
- kernel/bpf/verifier.c verifier integration for RET_PTR_TO_RINGBUF_MEM_OR_NULL / OBJ_RELEASE (covered in `verifier.md` Tier-3)
- kernel/bpf/helpers.c helper registration (covered in `bpf-core.md` Tier-3)
- include/linux/skmsg.h sk_msg redirection consumers (covered in `net/core/sock-map.md` Tier-3)
- libbpf user-space producer ringbuf API (covered in user-space companion if expanded)
- Implementation code
