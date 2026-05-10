---
title: "Tier-3: drivers/md/dm-bufio.c — block buffer cache (per-bdev metadata buffer cache + LRU eviction + transactional flush)"
tags: ["tier-3", "drivers-md", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

dm-bufio is the metadata-block buffer cache used by dm-thin, dm-cache, dm-integrity, dm-snap-persistent, etc. for stable + persistent metadata I/O — provides per-block 4KiB buffers with LRU eviction, dirty-tracking, transactional flush via `dm_bufio_write_dirty_buffers` for atomic-commit semantics. Critical for dm-targets that need to read/modify/write small metadata blocks frequently (e.g., B-tree updates in dm-thin metadata). Per-target dm_bufio_client owns its own cache; bufio shrinker integrates with kernel mm reclaim.

This Tier-3 covers `drivers/md/dm-bufio.c` (~2938 lines).

### Acceptance Criteria

- [ ] AC-1: Create dm_bufio_client; read 1000 blocks; verify cache hits on re-read.
- [ ] AC-2: Mark dirty + write_dirty_buffers: writes flushed; subsequent re-mount sees committed data.
- [ ] AC-3: LRU eviction: cache fills; oldest CLEAN evicted; DIRTY retained until flushed.
- [ ] AC-4: Async flush + barrier: large dirty list; async dispatch; barrier ensures all complete.
- [ ] AC-5: Per-client shrinker: memory pressure simulated; shrinker frees CLEAN buffers.
- [ ] AC-6: dm-thin metadata stress: 1M B-tree updates via dm-bufio; cache + flush correct.
- [ ] AC-7: dm-bufio with 32KB block size: large-metadata target works.
- [ ] AC-8: Crash mid-flush: kill -9; reboot; partial flush detected via dm-thin journal replay.
- [ ] AC-9: Multi-client: concurrent dm-thin + dm-cache; per-client caches isolated.
- [ ] AC-10: dm-bufio xfstests pass.

### Architecture

`DmBufioClient`:

```
struct DmBufioClient {
  bdev: KArc<BlockDevice>,
  block_size: u32,
  block_size_log2: u32,
  aux_size: u32,
  lock: Mutex<()>,
  cache: KArc<DmBufioCache>,
  n_buffers: [AtomicI32; LIST_SIZE],            // [CLEAN, DIRTY, READING, WRITING]
  max_buffers_per_client: u32,
  current_buffers: AtomicI32,
  min_n_buffers: u32,
  max_age_hz: u32,
  shrinker: KArc<Shrinker>,
  read_callback: Option<DmBufioReadCallback>,
  write_callback: Option<DmBufioWriteCallback>,
  free_callback: Option<DmBufioFreeCallback>,
  cleaner_thread: Option<KThread>,
  ...
}

struct DmBuffer {
  block: u64,
  data: KBox<[u8]>,                              // size == client.block_size
  state: AtomicU32,                              // STATE_*
  last_accessed: u64,                            // jiffies
  hold_count: AtomicI32,
  link: ListNode,
  hash_link: HlistNode,
  client: KWeak<DmBufioClient>,
  aux_data: KBox<[u8]>,                          // size == client.aux_size
}
```

`DmBufioClient::read(client, block, &buffer)`:
1. lock(client.lock).
2. b := __cache_lookup(client.cache, block).
3. If b:
   - If b.state == CLEAN: hold_count++; unlock; return b.
   - If b.state == ASYNC_READ: wait for completion; goto 2.
4. Else:
   - Alloc new b; b.block = block.
   - Insert in hash + LRU.
   - state = ASYNC_READ.
   - submit_bio(read).
5. unlock.
6. Wait for read completion.
7. Return b.

`DmBufioClient::write_dirty(client)`:
1. lock(client.lock).
2. For each b in client.dirty_lru:
   - Skip if hold_count > 0.
   - state = ASYNC_WRITE.
   - submit_bio(write, b).
3. unlock.
4. Wait for all in-flight writes.
5. submit_bio(flush+barrier, client.bdev).
6. Wait for flush.
7. lock; per-buffer state = CLEAN; move to clean LRU.
8. unlock.

`DmBufioClient::evict_blocks(client, max_age)`:
1. lock.
2. For each b in clean_lru (oldest first):
   - If b.hold_count > 0: skip.
   - If now - b.last_accessed > max_age: evict (free b).
   - Else: stop.
3. unlock.

### Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- dm-thin / dm-cache / dm-integrity / dm-snap-persistent (covered separately)
- Generic block-layer bdev (covered in `block/00-overview.md`)
- mm shrinker framework (covered in `mm/reclaim.md`)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dm_bufio_client` | per-target buffer cache | `drivers::md::bufio::DmBufioClient` |
| `struct dm_buffer` | per-block buffer | `DmBuffer` |
| `dm_bufio_client_create(...)` | per-bdev client create | `DmBufioClient::create` |
| `dm_bufio_client_destroy(client)` | client destroy | `DmBufioClient::destroy` |
| `dm_bufio_read(client, block, &buffer)` | per-block read | `DmBufioClient::read` |
| `dm_bufio_get(client, block, &buffer)` | per-block lookup-or-alloc (no read) | `DmBufioClient::get` |
| `dm_bufio_new(client, block, &buffer)` | per-block alloc + dirty | `DmBufioClient::new_block` |
| `dm_bufio_release(buffer)` | per-buffer release ref | `DmBuffer::release` |
| `dm_bufio_mark_buffer_dirty(buffer)` | mark dirty | `DmBuffer::mark_dirty` |
| `dm_bufio_mark_partial_buffer_dirty(...)` | partial mark | `DmBuffer::mark_partial_dirty` |
| `dm_bufio_write_dirty_buffers(client)` | flush all dirty | `DmBufioClient::write_dirty` |
| `dm_bufio_write_dirty_buffers_async(client)` | async flush | `DmBufioClient::write_dirty_async` |
| `dm_bufio_issue_flush(client)` | issue flush+barrier | `DmBufioClient::issue_flush` |
| `dm_bufio_forget(client, block)` | discard buffer | `DmBufioClient::forget` |
| `dm_bufio_release_move(buffer, new_block)` | move buffer to new block | `DmBuffer::release_move` |
| `dm_bufio_get_block_number(buffer)` | per-buffer block number | `DmBuffer::block_number` |
| `dm_bufio_get_block_data(buffer)` | per-buffer data pointer | `DmBuffer::data` |
| `dm_bufio_evict_blocks(client, max_age)` | LRU evict | `DmBufioClient::evict_blocks` |
| `__make_buffer_clean(buffer)` | per-buffer write + clean | `DmBuffer::make_clean` |
| `__check_watermark(client)` | per-client size check | `DmBufioClient::check_watermark` |

### compatibility contract

REQ-1: Per-target `dm_bufio_client`:
- `bdev` (block-device).
- `block_size` (typically 4KB).
- `aux_size` (per-buffer auxiliary data size; client-defined).
- `lock` (per-client mutex).
- `cache` (hash + LRU list).
- `n_buffers[]` (per-state count: clean, dirty, in-flight).
- `max_buffers_per_client` (hard cap).
- `current_buffers` (current count).
- `min_n_buffers` (low-water).
- `max_age_hz` (LRU max-age in jiffies).
- `shrinker` (integrates with mm reclaim).
- `read_callback` / `write_callback` / `free_callback` (per-client hooks).
- `cleaner_thread` (per-client background flush kthread; optional).

REQ-2: Per-buffer `dm_buffer`:
- `block` (logical block number).
- `data` (buffer-page-mapped data pointer).
- `state` (DM_BUFIO_STATE_*: CLEAN, DIRTY, ASYNC_READ, ASYNC_WRITE).
- `last_accessed` (jiffies for LRU).
- `hold_count` (refcount).
- `n_buffers_index` (per-state-list index).
- `link` (LRU list-node).
- `hash_link` (hash bucket-node).
- `aux_data` (client-defined; e.g., dm-thin uses for B-tree-node-validation).

REQ-3: Per-block read flow (`dm_bufio_read`):
1. Acquire client.lock.
2. Look up buffer in hash:
   - If found + CLEAN: return; ref incremented.
   - If found + ASYNC_READ: wait for completion.
   - If not found: alloc new; submit_bio; wait for completion.
3. unlock.
4. Return buffer.

REQ-4: Per-block dirty-tracking:
- `dm_bufio_mark_buffer_dirty(buffer)`:
  - Set state = DIRTY.
  - Move from clean-LRU to dirty-LRU.
  - n_buffers[clean]--; n_buffers[dirty]++.

REQ-5: Per-client flush:
- `dm_bufio_write_dirty_buffers`:
  - For each buffer in dirty-LRU:
    - submit write; wait for completion.
  - Issue flush+barrier to bdev.
- Synchronous wait; called at transaction-commit boundary.

REQ-6: Async flush:
- `dm_bufio_write_dirty_buffers_async`:
  - For each buffer: submit write asynchronously.
  - Caller may wait via `dm_bufio_issue_flush` for barrier.

REQ-7: LRU eviction:
- `__check_watermark(client)`:
  - If current_buffers > max_buffers_per_client:
    - Evict oldest CLEAN from LRU until below threshold.
    - DIRTY buffers cannot be evicted; flush first.

REQ-8: Per-buffer `aux_data`:
- Client-defined per-buffer aux data (e.g., dm-thin uses for B-tree-validator hooks: per-buffer is_valid + is_dirty + needs_check fields).

REQ-9: Per-client cleaner thread:
- Optional kthread for periodic flush.
- Triggers writeback if dirty buffer count > threshold.

REQ-10: Memory pressure integration:
- Per-client shrinker registered with mm.
- On-shrink: evict CLEAN buffers; ignore DIRTY.

REQ-11: bufio block size:
- 4KB / 8KB / 16KB / 32KB common.
- Per-client fixed at create.

REQ-12: Per-buffer hold_count:
- Incremented on get/read/new; decremented on release.
- Buffer not evictable when held.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `buffer_state_valid` | INVARIANT | per-buffer state ∈ {CLEAN, DIRTY, ASYNC_READ, ASYNC_WRITE}. |
| `hold_count_no_underflow` | INVARIANT | per-buffer hold_count ≥ 0. |
| `dirty_buffers_not_evicted` | INVARIANT | DIRTY buffers excluded from LRU eviction. |
| `cache_hash_consistent` | INVARIANT | per-block hash bucket entry matches LRU entry. |
| `n_buffers_count_match` | INVARIANT | n_buffers[state] == count of buffers in state-list. |

### Layer 2: TLA+

`drivers/md/bufio_state_machine.tla`:
- Per-buffer state ∈ {Free, Allocated, AsyncRead, Clean, Dirty, AsyncWrite, Released}.
- Properties:
  - `safety_only_one_async_op_per_buffer` — per-buffer at most one async op in flight.
  - `safety_dirty_persists_until_clean` — Dirty → Clean only via successful AsyncWrite.
  - `liveness_pending_eventually_completes` — every AsyncRead/AsyncWrite eventually completes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DmBufioClient::read` post: buffer returned with hold_count > 0 OR error | `DmBufioClient::read` |
| `DmBuffer::release` post: hold_count -= 1 | `DmBuffer::release` |
| `DmBufioClient::write_dirty` post: dirty buffers flushed; barrier issued | `DmBufioClient::write_dirty` |
| Per-cache hash + LRU consistency invariant | invariants on add/remove |

### Layer 4: Verus/Creusot functional

`Per-block: write to bufio + write_dirty_buffers + barrier → next read returns same data` semantic equivalence: per-block write durability after barrier holds across crash.

### hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-bufio-specific reinforcement:

- **Per-client max_buffers cap** — defense against unbounded cache growth.
- **DIRTY buffers excluded from LRU evict** — defense against dirty-data loss.
- **Per-buffer hold_count** — defense against UAF on buffer-in-use eviction.
- **submit_bio + barrier discipline** — defense against post-crash partial-flush.
- **Per-cache hash + LRU mutex** — defense against torn cache state.
- **Per-buffer state-machine atomic transitions** — defense against torn state.
- **Per-client shrinker integration** — defense against unbounded memory under pressure.
- **Cleaner-thread bounded by max_buffers** — defense against unbounded writeback.
- **read_callback / write_callback per-client** — defense against generic-buffer state misinterpretation.
- **Per-buffer aux_data isolated** — defense against client-aux corrupting bufio state.

