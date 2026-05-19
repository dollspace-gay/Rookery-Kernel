# Tier-3: fs/btrfs/space-info.c — btrfs space-info accounting + ENOSPC flushing

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/00-overview.md
upstream-paths:
  - fs/btrfs/space-info.c (~2258 lines)
  - fs/btrfs/space-info.h (struct btrfs_space_info, enum btrfs_flush_state)
  - fs/btrfs/block-rsv.c (consumer)
  - fs/btrfs/transaction.c (consumer)
-->

## Summary

btrfs **space-info** is the arbiter of how many bytes can be allocated for a given chunk-class. Per-`struct btrfs_space_info` is created per-class: `BTRFS_BLOCK_GROUP_DATA`, `_METADATA`, `_SYSTEM`, and optionally `_METADATA_REMAP`; in `MIXED_GROUPS` filesystems data+metadata share one space-info. Per-aggregate counters: `total_bytes` (sum of block-group lengths), `bytes_used` (committed extents), `bytes_pinned` (extents freed but not yet usable until current txn commits), `bytes_reserved` (allocated-but-not-yet-referenced extents), `bytes_may_use` (intent-to-allocate, paired with `block_rsv`), `bytes_readonly` (RO block-groups / `bytes_super`), `bytes_zone_unusable` (zoned-mode dead regions), `disk_used`/`disk_total` (raw-device-side, multiplied by RAID factor). Per-reservation flow: `reserve_bytes()` → fast-path bump `bytes_may_use` if `used + bytes <= total_bytes` OR `btrfs_can_overcommit(...)` (metadata-only). On shortage: enqueue `struct reserve_ticket` on `space_info->{tickets, priority_tickets}`, kick `btrfs_async_reclaim_{metadata,data}_space` workqueue → step through `enum btrfs_flush_state` (FLUSH_DELAYED_ITEMS_NR → FLUSH_DELALLOC → FLUSH_DELAYED_REFS → ALLOC_CHUNK → RUN_DELAYED_IPUTS → COMMIT_TRANS → RECLAIM_ZONES → RESET_ZONES → ALLOC_CHUNK_FORCE). Per-`wait_reserve_ticket` blocks (TASK_KILLABLE) until ticket->bytes == 0 or ticket->error set. Per-`maybe_fail_all_tickets` raises ENOSPC after flushing exhausted; per-`steal_from_global_rsv` last-resort for `BTRFS_RESERVE_FLUSH_ALL_STEAL` / `_FLUSH_EVICT`. Critical for: ENOSPC determinism, overcommit safety, transaction-abort recovery, zoned-mode reclaim, dynamic-reclaim threshold.

This Tier-3 covers `fs/btrfs/space-info.c` (~2258 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct btrfs_space_info` | per-class accounting + flushing | `SpaceInfo` |
| `struct reserve_ticket` | per-pending reservation | `ReserveTicket` |
| `btrfs_init_space_info()` | per-fs create SYSTEM/METADATA/DATA[/REMAP] | `SpaceInfo::init_fs` |
| `create_space_info()` | per-class allocate+sysfs+list-add | `SpaceInfo::create` |
| `create_space_info_sub_group()` | per-zoned sub_group (TREELOG/DATA_RELOC) | `SpaceInfo::create_sub_group` |
| `btrfs_add_bg_to_space_info()` | per-BG attach + counter bump | `SpaceInfo::add_block_group` |
| `btrfs_find_space_info()` | per-flag lookup | `SpaceInfo::find` |
| `btrfs_clear_space_info_full()` | per-add bg clear `full` | `SpaceInfo::clear_full` |
| `btrfs_update_space_info_chunk_size()` | per-update chunk_size hint | `SpaceInfo::update_chunk_size` |
| `calc_chunk_size()` | per-class+volume chunk-size policy | `SpaceInfo::calc_chunk_size` |
| `calc_effective_data_chunk_size()` | per-DATA effective chunk-size | `SpaceInfo::effective_data_chunk_size` |
| `calc_available_free_space()` | per-class overcommit budget | `SpaceInfo::available_free_space` |
| `check_can_overcommit()` / `can_overcommit()` | per-policy overcommit gate | `SpaceInfo::can_overcommit` |
| `btrfs_can_overcommit()` | per-callers wrapper | `SpaceInfo::can_overcommit_pub` |
| `btrfs_try_granting_tickets()` | per-wakeup eligible tickets | `SpaceInfo::try_granting_tickets` |
| `remove_ticket()` | per-ticket detach + wake | `SpaceInfo::remove_ticket` |
| `flush_space()` | per-state flush dispatcher | `SpaceInfo::flush_space` |
| `shrink_delalloc()` | per-FLUSH_DELALLOC[_WAIT,_FULL] | `SpaceInfo::shrink_delalloc` |
| `btrfs_calc_reclaim_metadata_size()` | per-metadata to-reclaim | `SpaceInfo::calc_reclaim_metadata_size` |
| `need_preemptive_reclaim()` | per-80% watermark check | `SpaceInfo::need_preemptive_reclaim` |
| `steal_from_global_rsv()` | per-EVICT/STEAL last-resort | `SpaceInfo::steal_from_global_rsv` |
| `maybe_fail_all_tickets()` | per-exhausted ENOSPC | `SpaceInfo::maybe_fail_all_tickets` |
| `do_async_reclaim_metadata_space()` | per-async loop body | `SpaceInfo::do_async_reclaim_metadata_space` |
| `btrfs_async_reclaim_metadata_space()` | per-WQ entry (metadata) | `SpaceInfo::async_reclaim_metadata_space` |
| `do_async_reclaim_data_space()` / `btrfs_async_reclaim_data_space()` | per-async loop (data) | `SpaceInfo::async_reclaim_data_space` |
| `btrfs_preempt_reclaim_metadata_space()` | per-preempt-flusher | `SpaceInfo::preempt_reclaim_metadata_space` |
| `btrfs_init_async_reclaim_work()` | per-fs WQ init | `SpaceInfo::init_async_reclaim_work` |
| `priority_reclaim_metadata_space()` | per-LIMIT/EVICT inline flush | `SpaceInfo::priority_reclaim_metadata_space` |
| `priority_reclaim_data_space()` | per-FREE_SPACE_INODE flush | `SpaceInfo::priority_reclaim_data_space` |
| `wait_reserve_ticket()` | per-block on ticket | `SpaceInfo::wait_reserve_ticket` |
| `handle_reserve_ticket()` | per-flush dispatch + wait | `SpaceInfo::handle_reserve_ticket` |
| `reserve_bytes()` | per-class fast-path + ticket | `SpaceInfo::reserve_bytes` |
| `btrfs_reserve_metadata_bytes()` | per-metadata reserve API | `SpaceInfo::reserve_metadata_bytes` |
| `btrfs_reserve_data_bytes()` | per-data reserve API | `SpaceInfo::reserve_data_bytes` |
| `btrfs_account_ro_block_groups_free_space()` | per-RO-bg free accounting | `SpaceInfo::account_ro_bgs_free_space` |
| `btrfs_dump_space_info()` / `_for_trans_abort()` | per-ENOSPC diagnostic | `SpaceInfo::dump` |
| `btrfs_return_free_space()` | per-release back-to-tickets | `SpaceInfo::return_free_space` |
| `btrfs_calc_reclaim_threshold()` | per-dynamic vs static threshold | `SpaceInfo::calc_reclaim_threshold` |
| `btrfs_reclaim_sweep()` | per-periodic sweep | `SpaceInfo::reclaim_sweep` |
| `btrfs_space_info_update_reclaimable()` | per-mark reclaimable | `SpaceInfo::update_reclaimable` |
| `btrfs_set_periodic_reclaim_ready()` | per-toggle periodic-reclaim | `SpaceInfo::set_periodic_reclaim_ready` |
| `enum btrfs_flush_state` | per-step flushing pipeline | shared |
| `enum btrfs_reserve_flush_enum` | per-caller flush policy | shared |

## Compatibility contract

REQ-1: struct btrfs_space_info (in-memory; per-class):
- fs_info: back-pointer to parent btrfs_fs_info.
- flags (u64): BTRFS_BLOCK_GROUP_TYPE_MASK ∈ {DATA, METADATA, SYSTEM, METADATA_REMAP, DATA|METADATA (mixed)}.
- subgroup_id: BTRFS_SUB_GROUP_PRIMARY | _TREELOG | _DATA_RELOC.
- parent / sub_group[BTRFS_SPACE_INFO_SUB_GROUP_MAX]: zoned-mode hierarchical layout.
- total_bytes (u64): Σ block_group->length for member BGs (REMAPPED with no identity-remap excluded).
- bytes_used (u64): Σ block_group->used.
- bytes_pinned (u64): freed-but-not-yet-usable extents (paired with EXTENT_PINNED in pinned_extents).
- bytes_reserved (u64): allocated extents not yet referenced.
- bytes_may_use (u64): intent-to-allocate reservations (block-rsv backed).
- bytes_readonly (u64): bytes_super + RO block-group lengths.
- bytes_zone_unusable (u64): zoned-mode unusable (already-written sequential regions).
- disk_used / disk_total (u64): RAID-factor-multiplied raw-device bytes.
- chunk_size (u64): policy hint for chunk-allocation (NOT a hard limit).
- full (bool): no unallocated extents left (set once ALLOC_CHUNK fails).
- lock (spinlock): protects counters + tickets + flush flag.
- groups_sem (rwsem): protects block_groups[BTRFS_NR_RAID_TYPES] + ro_bgs.
- block_groups[BTRFS_NR_RAID_TYPES]: per-raid lists.
- ro_bgs: list of RO block-groups.
- tickets / priority_tickets: pending reserve_ticket queues.
- tickets_id (u64): monotonic generation; bumped on any ticket completion.
- reclaim_size (u64): Σ pending ticket->bytes.
- flush (bool): async-reclaim worker scheduled.
- force_alloc: CHUNK_ALLOC_NO_FORCE | _FORCE | _LIMITED.
- clamp (u8): 1..8 — preempt-reclaim aggressiveness (powers-of-2 scaling).
- bg_reclaim_threshold: per-BG reclaim trigger (%).
- dynamic_reclaim / periodic_reclaim / periodic_reclaim_ready / reclaimable_bytes: per-sweep state.

REQ-2: struct reserve_ticket (on stack of waiting task; lives on space_info->tickets|priority_tickets):
- bytes (u64): remaining bytes to satisfy.
- error (int): 0 / -ENOSPC / -EINTR / abort-error.
- steal (bool): permitted to drain from global_rsv.
- list (list_head): queue linkage.
- wait (wait_queue_head_t): wakeup target.
- lock (spinlock): protects bytes/error transition.

REQ-3: btrfs_init_space_info(fs_info):
- disk_super = fs_info.super_copy; !btrfs_super_root → -EINVAL.
- features = btrfs_super_incompat_flags(disk_super); mixed = !!(features & MIXED_GROUPS).
- create_space_info(fs_info, BTRFS_BLOCK_GROUP_SYSTEM).
- If mixed: create_space_info(METADATA | DATA).
- Else: create_space_info(METADATA); create_space_info(DATA).
- If features & INCOMPAT_REMAP_TREE: create_space_info(METADATA_REMAP).
- Returns -ENOMEM on alloc failure; sysfs-add failures bubble up.

REQ-4: create_space_info(info, flags):
- kzalloc(struct btrfs_space_info, GFP_NOFS).
- init_space_info: init RAID lists, groups_sem, lock, flags & BLOCK_GROUP_TYPE_MASK, force_alloc=NO_FORCE, ro_bgs, tickets, priority_tickets, clamp=1, chunk_size via calc_chunk_size, subgroup_id=PRIMARY.
- If zoned: bg_reclaim_threshold = BTRFS_DEFAULT_ZONED_RECLAIM_THRESH (75).
- For zoned-mode: create_space_info_sub_group for BTRFS_SUB_GROUP_DATA_RELOC (DATA) or BTRFS_SUB_GROUP_TREELOG (METADATA).
- btrfs_sysfs_add_space_info_type.
- list_add(space_info.list, info.space_info).
- If DATA: info.data_sinfo = space_info.

REQ-5: calc_chunk_size(fs_info, flags):
- zoned ⟹ fs_info.zone_size.
- DATA ⟹ BTRFS_MAX_DATA_CHUNK_SIZE (10G).
- SYSTEM | METADATA_REMAP ⟹ SZ_32M.
- METADATA: total_rw_bytes > 50G ⟹ SZ_1G; else SZ_256M.

REQ-6: btrfs_add_bg_to_space_info(info, block_group):
- factor = btrfs_bg_type_to_factor(block_group.flags) (DUP/RAID1/RAID10 = 2; RAID0/single = 1; etc).
- spin_lock(space_info.lock):
  - If !(block_group.flags & BTRFS_BLOCK_GROUP_REMAPPED) ∨ block_group.identity_remap_count != 0:
    - total_bytes += block_group.length.
    - disk_total += block_group.length * factor.
  - bytes_used += block_group.used; disk_used += block_group.used * factor.
  - bytes_readonly += block_group.bytes_super.
  - btrfs_space_info_update_bytes_zone_unusable(zone_unusable).
  - If block_group.length > 0: full = false.
  - btrfs_try_granting_tickets(space_info).
- index = btrfs_bg_flags_to_raid_index; down_write(groups_sem); list_add_tail; up_write.

REQ-7: btrfs_find_space_info(info, flags):
- flags &= BTRFS_BLOCK_GROUP_TYPE_MASK.
- Iterate info.space_info; first entry with (found.flags & flags) non-zero → return.
- NULL on miss.

REQ-8: calc_available_free_space(space_info, flush):
- profile = SYSTEM ⟹ btrfs_system_alloc_profile; else btrfs_metadata_alloc_profile.
- has_per_profile = btrfs_get_per_profile_avail(fs_info, profile, &avail).
- If !has_per_profile: avail = atomic64_read(free_chunk_space); factor = btrfs_bg_type_to_factor(profile); avail /= factor; return 0 if zero.
- data_chunk_size = calc_effective_data_chunk_size.
- If avail <= data_chunk_size: return 0.
- avail -= data_chunk_size.
- If flush ∈ {FLUSH_ALL, FLUSH_ALL_STEAL}: avail >>= 6 (1/64th).
- Else: avail >>= 1 (1/2th).
- If zoned: ALIGN_DOWN(avail, zone_size).
- return avail.

REQ-9: check_can_overcommit(space_info, used, bytes, flush):
- avail = calc_available_free_space(space_info, flush).
- return (used + bytes < space_info.total_bytes + avail).

REQ-10: can_overcommit(space_info, used, bytes, flush) (internal):
- If space_info.flags & BTRFS_BLOCK_GROUP_DATA: return false (DATA never overcommits).
- return check_can_overcommit.

REQ-11: btrfs_can_overcommit(space_info, bytes, flush) (public):
- DATA ⟹ false.
- used = btrfs_space_info_used(space_info, true).
- return check_can_overcommit(space_info, used, bytes, flush).

REQ-12: btrfs_try_granting_tickets(space_info) — caller holds space_info.lock:
- Iterate priority_tickets first; then tickets with flush=FLUSH_ALL.
- For each head ticket:
  - used_after = used + ticket.bytes.
  - If used_after <= total_bytes ∨ can_overcommit(used, ticket.bytes, flush):
    - btrfs_space_info_update_bytes_may_use(space_info, ticket.bytes).
    - remove_ticket(space_info, ticket, 0).
    - tickets_id += 1.
    - used = used_after.
  - Else: break.

REQ-13: remove_ticket(space_info, ticket, error) — caller holds space_info.lock:
- If !list_empty(ticket.list): list_del_init; ASSERT(reclaim_size >= ticket.bytes); reclaim_size -= ticket.bytes.
- spin_lock(ticket.lock): if error ∧ ticket.bytes > 0: ticket.error = error; else: ticket.bytes = 0. wake_up(ticket.wait). spin_unlock.

REQ-14: flush_space(space_info, num_bytes, state, for_preempt) — dispatcher:
- FLUSH_DELAYED_ITEMS[_NR]: btrfs_join_transaction_nostart → btrfs_run_delayed_items_nr(trans, nr).
- FLUSH_DELALLOC[_WAIT|_FULL]: shrink_delalloc (FULL ⟹ num_bytes = U64_MAX).
- FLUSH_DELAYED_REFS[_NR]: btrfs_run_delayed_refs(trans, num_bytes-or-0).
- ALLOC_CHUNK[_FORCE]: btrfs_chunk_alloc(trans, space_info, profile, NO_FORCE|FORCE).
- RECLAIM_ZONES: zoned-only — btrfs_reclaim_sweep + btrfs_delete_unused_bgs + btrfs_reclaim_block_groups + commit_current_transaction.
- RUN_DELAYED_IPUTS: btrfs_run_delayed_iputs + btrfs_wait_on_delayed_iputs.
- COMMIT_TRANS: btrfs_commit_current_transaction.
- RESET_ZONES: btrfs_reset_unused_block_groups(space_info, num_bytes).
- default: -ENOSPC.
- trace_btrfs_flush_space(...).

REQ-15: shrink_delalloc(space_info, to_reclaim, wait_ordered, for_preempt):
- delalloc = percpu_counter_sum_positive(fs_info.delalloc_bytes); ordered = percpu_counter_sum_positive(ordered_bytes).
- If both 0: return.
- items = (to_reclaim == U64_MAX) ? U64_MAX : calc_reclaim_items_nr(to_reclaim) * 2; to_reclaim = max(to_reclaim, delalloc >> 3).
- If ordered > delalloc ∧ !for_preempt: wait_ordered = true.
- Loop up to 3 iterations:
  - nr_pages = min(delalloc, to_reclaim) >> PAGE_SHIFT.
  - btrfs_start_delalloc_roots(fs_info, nr_pages, true).
  - Wait for async_delalloc_pages to drain.
  - If wait_ordered ∧ !trans: btrfs_wait_ordered_roots(fs_info, items, NULL); else: schedule_timeout_killable(1) (break on signal).
  - For-preempt: break after one iteration.
  - If tickets empty: break.
  - Re-sample delalloc + ordered.

REQ-16: btrfs_calc_reclaim_metadata_size(space_info) — caller holds lock:
- avail = calc_available_free_space(FLUSH_ALL).
- used = btrfs_space_info_used(true).
- to_reclaim = reclaim_size.
- If total_bytes + avail < used: to_reclaim += used − (total_bytes + avail) (overage).
- return to_reclaim.

REQ-17: need_preemptive_reclaim(space_info) — caller holds lock:
- reclaim_size != 0: false (defer to async).
- thresh = 90% of total_bytes; if (used+reserved+global_rsv) >= thresh: false (too full to help).
- used = bytes_may_use + bytes_pinned.
- If global_rsv_size >= used: false.
- If used − global_rsv_size <= SZ_128M: false.
- thresh = calc_available_free_space(FLUSH_ALL); add total_bytes − (used+reserved+readonly+global_rsv) if positive; thresh >>= clamp.
- used = bytes_pinned.
- ordered = ordered_bytes >> 1; delalloc = delalloc_bytes.
- If ordered >= delalloc: used += delayed_refs_rsv.reserved + delayed_block_rsv.reserved.
- Else: used += bytes_may_use − global_rsv_size.
- Return (used >= thresh ∧ !fs_closing ∧ !REMOUNTING).

REQ-18: steal_from_global_rsv(space_info, ticket) — caller holds space_info.lock:
- !ticket.steal: false. global_rsv.space_info != space_info: false.
- min_bytes = mult_perc(global_rsv.size, 10) (keep ≥10%).
- If global_rsv.reserved < min_bytes + ticket.bytes: false.
- global_rsv.reserved -= ticket.bytes; if reserved < size: global_rsv.full = false.
- remove_ticket(0); tickets_id += 1; return true.

REQ-19: maybe_fail_all_tickets(space_info) — caller holds lock:
- tickets_id_snapshot = space_info.tickets_id.
- abort_error = BTRFS_FS_ERROR(fs_info).
- While !list_empty(tickets) ∧ tickets_id_snapshot == space_info.tickets_id:
  - ticket = head.
  - abort_error ⟹ remove_ticket(abort_error).
  - Else: if steal_from_global_rsv(ticket): return true; remove_ticket(-ENOSPC); btrfs_try_granting_tickets to wake smaller queued tickets.
- Return (tickets_id changed).

REQ-20: do_async_reclaim_metadata_space(space_info):
- spin_lock; to_reclaim = btrfs_calc_reclaim_metadata_size; !to_reclaim ⟹ flush=false; unlock; return.
- last_tickets_id = tickets_id; unlock.
- final_state = zoned ? RESET_ZONES : COMMIT_TRANS.
- flush_state = FLUSH_DELAYED_ITEMS_NR.
- Loop while flush_state <= final_state:
  - flush_space(to_reclaim, flush_state, false).
  - Re-lock; if list_empty(tickets): flush=false; return.
  - to_reclaim = btrfs_calc_reclaim_metadata_size.
  - If last_tickets_id == tickets_id: flush_state += 1.
  - Else: last_tickets_id = tickets_id; flush_state = FLUSH_DELAYED_ITEMS_NR; commit_cycles -= 1.
  - flush_state == FLUSH_DELALLOC_FULL ∧ !commit_cycles ⟹ skip (defer aggressive delalloc).
  - flush_state == ALLOC_CHUNK_FORCE ∧ !commit_cycles ⟹ skip (defer force-chunk).
  - flush_state > final_state:
    - commit_cycles += 1.
    - commit_cycles > 2: maybe_fail_all_tickets — true ⟹ flush_state = FLUSH_DELAYED_ITEMS_NR, commit_cycles -= 1; false ⟹ flush=false.
    - else: flush_state = FLUSH_DELAYED_ITEMS_NR.
  - Unlock; loop.

REQ-21: btrfs_async_reclaim_metadata_space(work):
- fs_info = container_of(work, btrfs_fs_info, async_reclaim_work).
- space_info = btrfs_find_space_info(fs_info, METADATA).
- do_async_reclaim_metadata_space(space_info).
- For i in 0..BTRFS_SPACE_INFO_SUB_GROUP_MAX: if sub_group[i]: do_async_reclaim_metadata_space.

REQ-22: btrfs_async_reclaim_data_space(work):
- fs_info = container_of(work, btrfs_fs_info, async_data_reclaim_work).
- do_async_reclaim_data_space(fs_info.data_sinfo) and sub-groups.

REQ-23: do_async_reclaim_data_space(space_info):
- spin_lock; if !tickets: flush=false; return.
- last_tickets_id = tickets_id; unlock.
- Phase-1: while !space_info.full: flush_space(U64_MAX, ALLOC_CHUNK_FORCE, false); spin_lock; if !tickets: return; if BTRFS_FS_ERROR: aborted_fs (fail-all + bail).
- Phase-2: for state in data_flush_states {FLUSH_DELALLOC_FULL, RUN_DELAYED_IPUTS, COMMIT_TRANS, RECLAIM_ZONES, RESET_ZONES, ALLOC_CHUNK_FORCE}:
  - flush_space(U64_MAX, state, false).
  - Re-lock; if !tickets: return.
  - If last_tickets_id == tickets_id: state += 1; else: last_tickets_id = tickets_id; state = 0.
  - state >= ARRAY_SIZE: if full: maybe_fail_all_tickets ⟹ state=0 else flush=false; else state=0.
- aborted_fs: maybe_fail_all_tickets; flush=false.

REQ-24: btrfs_preempt_reclaim_metadata_space(work):
- Pre-emptive flusher; runs while need_preemptive_reclaim(space_info):
  - Pick flush: delalloc_size > block_rsv_size ⟹ FLUSH_DELALLOC; bytes_pinned > delayed_block+refs ⟹ COMMIT_TRANS; delayed_block > delayed_refs ⟹ FLUSH_DELAYED_ITEMS_NR; else FLUSH_DELAYED_REFS_NR.
  - to_reclaim >>= 2; if 0: btrfs_calc_insert_metadata_size(1).
  - flush_space(to_reclaim, flush, true).
  - cond_resched.
- Loops once + still satisfied ⟹ clamp = max(1, clamp − 1).

REQ-25: priority_reclaim_metadata_space(space_info, ticket, states, states_nr):
- If is_ticket_served(ticket): return.
- spin_lock; to_reclaim = btrfs_calc_reclaim_metadata_size; unlock.
- For state in states[0..states_nr]: flush_space(to_reclaim, state, false). If is_ticket_served: return.
- spin_lock; if BTRFS_FS_ERROR: remove_ticket(error). Else if !steal_from_global_rsv(ticket): remove_ticket(-ENOSPC). btrfs_try_granting_tickets; unlock.

REQ-26: priority_reclaim_data_space(space_info, ticket):
- If is_ticket_served(ticket): return.
- spin_lock; while !space_info.full: unlock; flush_space(U64_MAX, ALLOC_CHUNK_FORCE, false); if is_ticket_served: return; lock.
- remove_ticket(-ENOSPC); btrfs_try_granting_tickets; unlock.

REQ-27: wait_reserve_ticket(space_info, ticket):
- DEFINE_WAIT(wait).
- spin_lock(ticket.lock).
- While ticket.bytes > 0 ∧ ticket.error == 0:
  - prepare_to_wait_event(ticket.wait, TASK_KILLABLE).
  - unlock(ticket.lock).
  - If signal (ret): spin_lock(space_info.lock); remove_ticket(-EINTR); unlock; return.
  - schedule().
  - finish_wait; spin_lock(ticket.lock).
- unlock(ticket.lock).

REQ-28: handle_reserve_ticket(space_info, ticket, start_ns, orig_bytes, flush):
- FLUSH_DATA | FLUSH_ALL | FLUSH_ALL_STEAL: wait_reserve_ticket.
- FLUSH_LIMIT: priority_reclaim_metadata_space(priority_flush_states {FLUSH_DELAYED_ITEMS_NR, FLUSH_DELAYED_ITEMS, RESET_ZONES, ALLOC_CHUNK}).
- FLUSH_EVICT: priority_reclaim_metadata_space(evict_flush_states {FLUSH_DELAYED_ITEMS_NR, _ITEMS, _REFS_NR, _REFS, FLUSH_DELALLOC, _WAIT, _FULL, ALLOC_CHUNK, COMMIT_TRANS, RESET_ZONES}).
- FLUSH_FREE_SPACE_INODE: priority_reclaim_data_space.
- default: ASSERT(0).
- ret = ticket.error; ASSERT(list_empty(ticket.list)); ASSERT(!(bytes==0 ∧ error)).
- trace_btrfs_reserve_ticket; return ret.

REQ-29: reserve_bytes(space_info, orig_bytes, flush):
- ASSERT(orig_bytes != 0).
- If current.journal_info: ASSERT flush ∉ {FLUSH_ALL, _ALL_STEAL, _EVICT}.
- async_work = (flush == FLUSH_DATA) ? async_data_reclaim_work : async_reclaim_work.
- spin_lock(space_info.lock).
- used = btrfs_space_info_used(true).
- pending_tickets = (normal_flush ∨ NO_FLUSH) ? (!empty(tickets) ∨ !empty(priority_tickets)) : !empty(priority_tickets).
- Fast-path: !pending_tickets ∧ (used + orig_bytes <= total_bytes ∨ can_overcommit) ⟹ bytes_may_use += orig_bytes; ret=0.
- FLUSH_EMERGENCY fallback: used -= bytes_may_use; if (used + orig_bytes <= total_bytes): bytes_may_use += orig_bytes; ret=0.
- If ret != 0 ∧ can_ticket(flush):
  - Setup ticket on stack (bytes, error=0, wait, lock, steal = can_steal(flush)).
  - reclaim_size += orig_bytes.
  - If FLUSH_ALL | _ALL_STEAL | _DATA:
    - list_add_tail(ticket, tickets).
    - If !flush: maybe_clamp_preempt; flush=true; queue_work(async_work).
  - Else: list_add_tail(ticket, priority_tickets).
- Else if ret==0 ∧ METADATA: if !LOG_RECOVERING ∧ !work_busy(preempt_reclaim_work) ∧ need_preemptive_reclaim: queue_work(preempt_reclaim_work).
- spin_unlock.
- If ret==0 ∨ !can_ticket(flush): return ret.
- return handle_reserve_ticket(space_info, &ticket, start_ns, orig_bytes, flush).

REQ-30: btrfs_reserve_metadata_bytes(space_info, orig_bytes, flush):
- ret = reserve_bytes; -ENOSPC ⟹ trace + dump (if ENOSPC_DEBUG). Return ret.

REQ-31: btrfs_reserve_data_bytes(space_info, bytes, flush):
- ASSERT flush ∈ {FLUSH_DATA, FREE_SPACE_INODE, NO_FLUSH}.
- ASSERT !current.journal_info ∨ flush != FLUSH_DATA.
- ret = reserve_bytes; -ENOSPC ⟹ trace + dump (if ENOSPC_DEBUG). Return ret.

REQ-32: btrfs_return_free_space(space_info, len) — caller holds space_info.lock:
- If global_rsv.space_info == space_info ∧ !global_rsv.full:
  - to_add = min(len, global_rsv.size − global_rsv.reserved).
  - global_rsv.reserved += to_add; btrfs_space_info_update_bytes_may_use(to_add); full = (reserved >= size).
  - len -= to_add.
- If len: btrfs_try_granting_tickets.

REQ-33: btrfs_account_ro_block_groups_free_space(sinfo):
- data_race(list_empty(ro_bgs)) ⟹ 0.
- spin_lock; Σ (bg.length − bg.used) * factor over RO BGs; spin_unlock; return.

REQ-34: maybe_clamp_preempt(space_info):
- ordered = sum(ordered_bytes); delalloc = sum(delalloc_bytes).
- If ordered < delalloc: clamp = min(clamp + 1, 8).

REQ-35: Periodic / dynamic reclaim:
- calc_dynamic_reclaim_threshold(space_info): unalloc = free_chunk_space; target = 10 * effective_data_chunk_size; want = max(target − unalloc, 0); unused = total − used; if unused < data_chunk_size: return 0; else return calc_pct_ratio(want, target).
- btrfs_calc_reclaim_threshold: dynamic_reclaim ⟹ dynamic else bg_reclaim_threshold.
- is_reclaim_urgent: free_chunk_space < data_chunk_size.
- do_reclaim_sweep(space_info, raid): iterate block_groups[raid]; mark bg.reclaim_mark + bg.used < thresh_pct ⟹ btrfs_mark_bg_to_reclaim; second pass if urgent and nothing matched.
- btrfs_reclaim_sweep(fs_info): per-space_info → per-raid do_reclaim_sweep.
- btrfs_space_info_update_reclaimable(space_info, bytes): reclaimable_bytes += bytes; if >= chunk_sz: btrfs_set_periodic_reclaim_ready(true).
- btrfs_set_periodic_reclaim_ready(space_info, ready): toggles periodic_reclaim_ready; on false: reclaimable_bytes = 0.

REQ-36: btrfs_dump_space_info(info, bytes, dump_block_groups) / __btrfs_dump_space_info / btrfs_dump_space_info_for_trans_abort:
- Print total, used, pinned, reserved, may_use, readonly, zone_unusable; sub_group_id; full state.
- dump_global_block_rsv prints rsv_name: size/reserved for global_block_rsv, trans_block_rsv, chunk_block_rsv, remap_block_rsv, delayed_block_rsv, delayed_refs_rsv.

## Acceptance Criteria

- [ ] AC-1: btrfs_init_space_info on non-mixed FS creates exactly {SYSTEM, METADATA, DATA} (+ METADATA_REMAP if REMAP_TREE incompat); on mixed FS creates {SYSTEM, DATA|METADATA}.
- [ ] AC-2: btrfs_add_bg_to_space_info bumps total_bytes, bytes_used, disk_used (× RAID factor), bytes_readonly (block_group.bytes_super), and clears `full` if length > 0.
- [ ] AC-3: btrfs_can_overcommit on a DATA space-info returns false unconditionally.
- [ ] AC-4: btrfs_can_overcommit on METADATA returns (used + bytes < total_bytes + calc_available_free_space(flush)).
- [ ] AC-5: reserve_bytes fast-path with sufficient space + no pending tickets bumps bytes_may_use and returns 0 without enqueueing a ticket.
- [ ] AC-6: reserve_bytes with insufficient space and FLUSH_ALL enqueues ticket on space_info.tickets, queues async_reclaim_work, sets space_info.flush = true.
- [ ] AC-7: handle_reserve_ticket(FLUSH_ALL) blocks in wait_reserve_ticket TASK_KILLABLE; signal causes -EINTR via remove_ticket and exits cleanly.
- [ ] AC-8: btrfs_try_granting_tickets satisfies head ticket when used_after <= total_bytes; wakes waiter; tickets_id increments; reclaim_size decrements by ticket.bytes.
- [ ] AC-9: flush_space(FLUSH_DELALLOC_FULL) sets num_bytes = U64_MAX before calling shrink_delalloc.
- [ ] AC-10: do_async_reclaim_metadata_space advances flush_state when tickets_id unchanged; resets to FLUSH_DELAYED_ITEMS_NR when tickets_id changed; commits ≤ 2 cycles before maybe_fail_all_tickets.
- [ ] AC-11: priority_reclaim_metadata_space(FLUSH_EVICT) cycles evict_flush_states; on exhaustion attempts steal_from_global_rsv else remove_ticket(-ENOSPC).
- [ ] AC-12: steal_from_global_rsv refuses when (global_rsv.reserved < mult_perc(global_rsv.size, 10) + ticket.bytes).
- [ ] AC-13: maybe_fail_all_tickets returns true when at least one ticket was failed (tickets_id changed); subsequent retry can resume from FLUSH_DELAYED_ITEMS_NR.
- [ ] AC-14: btrfs_return_free_space refills global_rsv to full before granting tickets when global_rsv.space_info == self.
- [ ] AC-15: btrfs_account_ro_block_groups_free_space sums (length − used) × factor only across `ro_bgs` with bg.ro == true.
- [ ] AC-16: do_async_reclaim_data_space first drains ALLOC_CHUNK_FORCE until `full`, then steps through data_flush_states; on BTRFS_FS_ERROR fail-all + bail (aborted_fs).
- [ ] AC-17: btrfs_calc_reclaim_threshold returns dynamic value when dynamic_reclaim set, else bg_reclaim_threshold (75 for zoned default).

## Architecture

```
struct SpaceInfo {
  fs_info: *FsInfo,
  flags: u64,                                    // BLOCK_GROUP_TYPE_MASK
  subgroup_id: u8,                               // PRIMARY|TREELOG|DATA_RELOC
  parent: *SpaceInfo,
  sub_group: [*SpaceInfo; BTRFS_SPACE_INFO_SUB_GROUP_MAX],

  // Counters (lock-protected)
  total_bytes: u64,
  bytes_used: u64,
  bytes_pinned: u64,
  bytes_reserved: u64,
  bytes_may_use: u64,
  bytes_readonly: u64,
  bytes_zone_unusable: u64,
  disk_total: u64,
  disk_used: u64,

  // Policy
  chunk_size: u64,
  force_alloc: u8,
  full: bool,
  clamp: u8,
  bg_reclaim_threshold: u32,
  dynamic_reclaim: bool,
  periodic_reclaim: bool,
  periodic_reclaim_ready: bool,
  reclaimable_bytes: s64,

  // Synchronization
  lock: Spinlock,
  groups_sem: RwSemaphore,

  // Members
  block_groups: [List<BlockGroup>; BTRFS_NR_RAID_TYPES],
  ro_bgs: List<BlockGroup>,

  // Tickets
  tickets: List<ReserveTicket>,
  priority_tickets: List<ReserveTicket>,
  tickets_id: u64,
  reclaim_size: u64,
  flush: bool,

  list: ListLink,                                // fs_info.space_info
}

struct ReserveTicket {
  bytes: u64,
  error: i32,
  steal: bool,
  list: ListLink,
  wait: WaitQueue,
  lock: Spinlock,
}
```

`SpaceInfo::reserve_bytes(space_info, orig_bytes, flush) -> i32`:
1. ASSERT(orig_bytes != 0).
2. /* Deadlock guards */
3. if current.journal_info ≠ NULL: ASSERT(flush ∉ {FLUSH_ALL, _ALL_STEAL, _EVICT}).
4. async_work = (flush == FLUSH_DATA) ? async_data_reclaim_work : async_reclaim_work.
5. spin_lock(space_info.lock).
6. used = SpaceInfo::used(space_info, true).
7. pending_tickets =
   - if is_normal_flushing(flush) ∨ flush == NO_FLUSH: !empty(tickets) ∨ !empty(priority_tickets).
   - else: !empty(priority_tickets).
8. /* Fast-path */
9. if !pending_tickets ∧ (used + orig_bytes <= total_bytes ∨ SpaceInfo::can_overcommit(space_info, used, orig_bytes, flush)):
   - bytes_may_use += orig_bytes.
   - ret = 0.
10. /* FLUSH_EMERGENCY backstop */
11. if ret ≠ 0 ∧ flush == FLUSH_EMERGENCY:
    - used -= bytes_may_use.
    - if used + orig_bytes <= total_bytes: bytes_may_use += orig_bytes; ret = 0.
12. /* Slow-path: enqueue ticket */
13. if ret ≠ 0 ∧ can_ticket(flush):
    - ticket = ReserveTicket{ bytes: orig_bytes, error: 0, steal: can_steal(flush), .. }.
    - reclaim_size += orig_bytes.
    - if flush ∈ {FLUSH_ALL, FLUSH_ALL_STEAL, FLUSH_DATA}:
      - list_add_tail(ticket.list, tickets).
      - if !space_info.flush:
        - maybe_clamp_preempt.
        - space_info.flush = true.
        - queue_work(system_dfl_wq, async_work).
    - else:
      - list_add_tail(ticket.list, priority_tickets).
14. /* Trigger preempt-reclaim if we just succeeded but pressure is high */
15. else if ret == 0 ∧ flags & METADATA:
    - if !LOG_RECOVERING ∧ !work_busy(preempt_reclaim_work) ∧ SpaceInfo::need_preemptive_reclaim:
      - queue_work(system_dfl_wq, preempt_reclaim_work).
16. spin_unlock(space_info.lock).
17. if ret == 0 ∨ !can_ticket(flush): return ret.
18. return SpaceInfo::handle_reserve_ticket(space_info, &ticket, start_ns, orig_bytes, flush).

`SpaceInfo::try_granting_tickets(space_info)` — lock held:
1. head = priority_tickets; flush = NO_FLUSH.
2. used = SpaceInfo::used(space_info, true).
3. Loop:
   - while !empty(head):
     - ticket = first(head).
     - used_after = used + ticket.bytes.
     - if used_after <= total_bytes ∨ SpaceInfo::can_overcommit(space_info, used, ticket.bytes, flush):
       - bytes_may_use += ticket.bytes.
       - SpaceInfo::remove_ticket(ticket, 0).
       - tickets_id += 1.
       - used = used_after.
     - else: break.
   - if head == priority_tickets: head = tickets; flush = FLUSH_ALL; goto loop.

`SpaceInfo::flush_space(space_info, num_bytes, state, for_preempt)`:
1. match state:
   - FLUSH_DELAYED_ITEMS[_NR] ⟹ run delayed inode-items.
   - FLUSH_DELALLOC[_WAIT|_FULL] ⟹ shrink_delalloc.
   - FLUSH_DELAYED_REFS[_NR] ⟹ run delayed refs.
   - ALLOC_CHUNK[_FORCE] ⟹ btrfs_chunk_alloc.
   - RECLAIM_ZONES ⟹ btrfs_reclaim_sweep + delete_unused_bgs + reclaim_block_groups + commit.
   - RUN_DELAYED_IPUTS ⟹ btrfs_run_delayed_iputs + wait.
   - COMMIT_TRANS ⟹ btrfs_commit_current_transaction.
   - RESET_ZONES ⟹ btrfs_reset_unused_block_groups.
   - default ⟹ -ENOSPC.
2. trace_btrfs_flush_space.

`SpaceInfo::do_async_reclaim_metadata_space(space_info)`:
1. final_state = zoned ? RESET_ZONES : COMMIT_TRANS.
2. spin_lock; to_reclaim = SpaceInfo::calc_reclaim_metadata_size; !to_reclaim ⟹ flush=false; return.
3. last_tickets_id = tickets_id; unlock.
4. flush_state = FLUSH_DELAYED_ITEMS_NR.
5. Loop:
   - SpaceInfo::flush_space(to_reclaim, flush_state, false).
   - spin_lock.
   - if empty(tickets): flush=false; return.
   - to_reclaim = SpaceInfo::calc_reclaim_metadata_size.
   - if last_tickets_id == tickets_id: flush_state += 1.
   - else: last_tickets_id = tickets_id; flush_state = FLUSH_DELAYED_ITEMS_NR; commit_cycles -= 1.
   - if flush_state == FLUSH_DELALLOC_FULL ∧ commit_cycles == 0: flush_state += 1.
   - if flush_state == ALLOC_CHUNK_FORCE ∧ commit_cycles == 0: flush_state += 1.
   - if flush_state > final_state:
     - commit_cycles += 1.
     - if commit_cycles > 2:
       - if SpaceInfo::maybe_fail_all_tickets: flush_state = FLUSH_DELAYED_ITEMS_NR; commit_cycles -= 1.
       - else: flush=false.
     - else: flush_state = FLUSH_DELAYED_ITEMS_NR.
   - spin_unlock.

`SpaceInfo::handle_reserve_ticket(space_info, ticket, start_ns, orig_bytes, flush) -> i32`:
1. match flush:
   - FLUSH_DATA | FLUSH_ALL | FLUSH_ALL_STEAL ⟹ SpaceInfo::wait_reserve_ticket.
   - FLUSH_LIMIT ⟹ SpaceInfo::priority_reclaim_metadata_space(priority_flush_states).
   - FLUSH_EVICT ⟹ SpaceInfo::priority_reclaim_metadata_space(evict_flush_states).
   - FLUSH_FREE_SPACE_INODE ⟹ SpaceInfo::priority_reclaim_data_space.
2. ret = ticket.error; ASSERT(list_empty(ticket.list)); ASSERT(!(ticket.bytes == 0 ∧ ticket.error)).
3. trace_btrfs_reserve_ticket; return ret.

`SpaceInfo::wait_reserve_ticket(space_info, ticket)`:
1. spin_lock(ticket.lock).
2. while ticket.bytes > 0 ∧ ticket.error == 0:
   - prepare_to_wait_event(ticket.wait, TASK_KILLABLE).
   - spin_unlock(ticket.lock).
   - if signal-pending:
     - spin_lock(space_info.lock).
     - SpaceInfo::remove_ticket(ticket, -EINTR).
     - spin_unlock(space_info.lock).
     - return.
   - schedule().
   - finish_wait.
   - spin_lock(ticket.lock).
3. spin_unlock(ticket.lock).

`SpaceInfo::maybe_fail_all_tickets(space_info) -> bool` — lock held:
1. snapshot_id = tickets_id.
2. abort_error = BTRFS_FS_ERROR.
3. while !empty(tickets) ∧ snapshot_id == tickets_id:
   - ticket = first(tickets).
   - if abort_error: SpaceInfo::remove_ticket(ticket, abort_error).
   - else:
     - if SpaceInfo::steal_from_global_rsv(ticket): return true.
     - SpaceInfo::remove_ticket(ticket, -ENOSPC).
     - SpaceInfo::try_granting_tickets.
4. return (snapshot_id != tickets_id).

`SpaceInfo::btrfs_async_reclaim_metadata_space(work)`:
1. fs_info = container_of(work, FsInfo, async_reclaim_work).
2. space_info = SpaceInfo::find(fs_info, BTRFS_BLOCK_GROUP_METADATA).
3. SpaceInfo::do_async_reclaim_metadata_space(space_info).
4. for i in 0..BTRFS_SPACE_INFO_SUB_GROUP_MAX:
   - if space_info.sub_group[i]: SpaceInfo::do_async_reclaim_metadata_space(space_info.sub_group[i]).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `counters_non_negative` | INVARIANT | per-space_info: total_bytes ≥ bytes_used + bytes_reserved + bytes_pinned + bytes_may_use + bytes_readonly + bytes_zone_unusable after every lock-protected transition; or overcommitted by ≤ avail. |
| `data_never_overcommits` | INVARIANT | per-can_overcommit: flags & BTRFS_BLOCK_GROUP_DATA ⟹ false. |
| `ticket_bytes_zero_implies_no_error` | INVARIANT | per-remove_ticket / handle_reserve_ticket: !(ticket.bytes == 0 ∧ ticket.error). |
| `tickets_id_monotone` | INVARIANT | per-try_granting_tickets / steal_from_global_rsv: tickets_id strictly increases on success. |
| `reclaim_size_balance` | INVARIANT | per-remove_ticket: reclaim_size ≥ ticket.bytes prior to decrement. |
| `journal_info_no_blocking_flush` | INVARIANT | per-reserve_bytes: current.journal_info ⟹ flush ∉ {FLUSH_ALL, _ALL_STEAL, _EVICT}. |
| `global_rsv_min_10pct` | INVARIANT | per-steal_from_global_rsv: post-steal global_rsv.reserved ≥ mult_perc(global_rsv.size, 10). |
| `flush_state_progress` | INVARIANT | per-do_async_reclaim_metadata_space: every iteration either advances flush_state, resets it on ticket-progress, or terminates via maybe_fail_all_tickets after ≤ 2 commit_cycles. |
| `fast_path_no_ticket_enqueue` | INVARIANT | per-reserve_bytes: ret == 0 fast-path ⟹ ticket not on any list. |
| `wait_ticket_killable` | INVARIANT | per-wait_reserve_ticket: schedule() preceded by prepare_to_wait_event(TASK_KILLABLE). |
| `try_granting_under_lock` | INVARIANT | per-try_granting_tickets: lockdep_assert_held(space_info.lock). |
| `dump_consistent` | INVARIANT | per-dump_space_info: holds space_info.lock around counter read. |

### Layer 2: TLA+

`fs/btrfs/space-info.tla`:
- Per-space_info FSM: counters + ticket queues + flush-state.
- Actions: reserve_bytes (fast-path | enqueue), flush_space (per-state), try_granting_tickets, remove_ticket, maybe_fail_all_tickets, steal_from_global_rsv, btrfs_add_bg_to_space_info, btrfs_return_free_space.
- Properties:
  - `safety_no_double_accounting` — per-state-transition: Σ bytes_* invariant under add-bg / release / reserve / unreserve cycles.
  - `safety_ticket_or_succeed` — per-reserve_bytes: every call ends in either fast-path-grant, enqueue (handled by handle_reserve_ticket), or NO_FLUSH/EMERGENCY ret.
  - `safety_metadata_overcommit_bounded` — per-can_overcommit: METADATA overcommit ≤ calc_available_free_space.
  - `safety_data_never_overcommit` — per-can_overcommit: DATA ⟹ false.
  - `liveness_async_reclaim_terminates` — per-do_async_reclaim_*: bounded by (final_state − FLUSH_DELAYED_ITEMS_NR) + 2 commit_cycles before maybe_fail_all_tickets.
  - `liveness_ticket_woken_or_errored` — per-ticket: eventually ticket.bytes == 0 ∨ ticket.error != 0.
  - `liveness_priority_before_normal` — per-try_granting_tickets: priority_tickets head drained before tickets head.
  - `liveness_preempt_clamp_bounded` — per-preempt-reclaim: clamp ∈ [1, 8].

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SpaceInfo::reserve_bytes` post: ret == 0 ⟹ bytes_may_use bumped under lock; ret == -ENOSPC ⟹ ticket handled or NO_FLUSH | `SpaceInfo::reserve_bytes` |
| `SpaceInfo::try_granting_tickets` post: granted ⟹ tickets_id increased ∧ bytes_may_use increased by ticket.bytes ∧ reclaim_size decreased | `SpaceInfo::try_granting_tickets` |
| `SpaceInfo::flush_space` post: tracepoint emitted; underlying flush operation invoked iff state valid | `SpaceInfo::flush_space` |
| `SpaceInfo::can_overcommit` post: DATA ⟹ false; else check_can_overcommit | `SpaceInfo::can_overcommit` |
| `SpaceInfo::add_block_group` post: total_bytes += bg.length ∧ disk_total += bg.length * factor (unless REMAPPED-no-identity) ∧ bytes_used += bg.used ∧ bytes_readonly += bg.bytes_super ∧ full = false if length > 0 | `SpaceInfo::add_block_group` |
| `SpaceInfo::find` post: NULL ∨ (found.flags & query_flags) ≠ 0 | `SpaceInfo::find` |
| `SpaceInfo::steal_from_global_rsv` post: true ⟹ global_rsv.reserved -= ticket.bytes ∧ ticket removed; false ⟹ no mutation | `SpaceInfo::steal_from_global_rsv` |
| `SpaceInfo::maybe_fail_all_tickets` post: true ⟹ ≥ 1 ticket removed (success-via-steal or -ENOSPC) | `SpaceInfo::maybe_fail_all_tickets` |
| `SpaceInfo::wait_reserve_ticket` post: returns iff (ticket.bytes == 0 ∨ ticket.error != 0 ∨ -EINTR via remove_ticket) | `SpaceInfo::wait_reserve_ticket` |
| `SpaceInfo::do_async_reclaim_metadata_space` post: terminates with flush = false ∨ tickets empty | `SpaceInfo::do_async_reclaim_metadata_space` |
| `SpaceInfo::return_free_space` post: global_rsv refilled toward full first when global_rsv.space_info == space_info; remainder feeds try_granting_tickets | `SpaceInfo::return_free_space` |

### Layer 4: Verus/Creusot functional

`Per-reserve flow: reserve_bytes (fast-path ∨ enqueue ticket + kick async/preempt worker) → handle_reserve_ticket (wait or priority-flush) → async/preempt worker steps flush_space through FLUSH_DELAYED_ITEMS_NR..final_state and calls try_granting_tickets after each progress event → grant or maybe_fail_all_tickets (steal or -ENOSPC); on release-path return_free_space refills global_rsv then grants tickets` semantic equivalence: per-`Documentation/filesystems/btrfs.rst` and per-`fs/btrfs/space-info.c` HOWTO comment block (lines 21-183).

## Hardening

(Inherits row-1 features from `fs/btrfs/00-overview.md` § Hardening.)

Space-info reinforcement:

- **Per-data-never-overcommits invariant** — defense against per-DATA ENOSPC-via-overcommit data-loss.
- **Per-global_rsv ≥ 10% floor on steal** — defense against per-emergency-pool exhaustion.
- **Per-journal_info no-blocking-flush ASSERT** — defense against per-transaction-commit deadlock.
- **Per-ticket TASK_KILLABLE + -EINTR removal** — defense against per-uninterruptible-hang on ENOSPC.
- **Per-FLUSH_EMERGENCY pinned-vs-may_use accounting** — defense against per-abort-spiral with stuck reservations.
- **Per-commit_cycles ≤ 2 then fail-tickets** — defense against per-livelock on terminal ENOSPC.
- **Per-need_preemptive_reclaim clamp 1..8** — defense against per-runaway preempt-flusher.
- **Per-can_overcommit divisor (1/2 vs 1/64 by flush state)** — defense against per-aggressive overcommit when caller cannot flush.
- **Per-bytes_zone_unusable accounted in `used`** — defense against per-zoned-mode incorrect overcommit on sequentially-dead regions.
- **Per-fs-error abort_error short-circuits all tickets** — defense against per-readonly-after-abort phantom reservations.
- **Per-tickets_id monotonic** — defense against per-double-wake race.
- **Per-remove_ticket holds space_info.lock + ticket.lock** — defense against per-concurrent-error race.
- **Per-priority_tickets drained before tickets** — defense against per-priority-inversion (FLUSH_LIMIT/EVICT starvation).

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
- **Ticket-handler PAX_REFCOUNT** — saturating refcount on `reserve_ticket` references to block per-ticket wake-after-free when async-reclaim races signal-driven `-EINTR` cleanup.
- **btrfs_can_overcommit() gating** — overcommit decision is locked behind `space_info.lock`, with DATA short-circuited to false; PAX_REFCOUNT prevents stale `space_info` deref during sysfs sweep.
- **kCFI on flush-state dispatcher** — `flush_space` switch dispatches to delayed-items, delalloc, refs, chunk-alloc, ipuTs, commit, zone-reset; each callee enforced via kCFI signatures so a corrupted `flush_state` cannot redirect to arbitrary text.
- **PAX_MEMORY_SANITIZE on ticket teardown** — on-stack ticket struct includes `ticket.lock` + `ticket.bytes`; zeroing on remove guards against information-leak via stack reuse for subsequent reserve callers.
- **GRKERNSEC_HIDESYM on btrfs_dump_space_info** — ENOSPC diagnostics never leak `fs_info` / `space_info` kernel pointers into dmesg.
- **GRKERNSEC_PROC_USER on sysfs space_info nodes** — `total_bytes`/`bytes_may_use` counters restricted to root + fs-admin to deny user-space ENOSPC oracle.
- **PAX_REFCOUNT on space_info.list** — saturating add/release across `btrfs_init_space_info` and unmount-path teardown.

Per-doc rationale: space-info is the per-class accounting and ENOSPC-flushing arbiter; reserve tickets on caller stacks, flush-state dispatch tables, and overcommit budgeting all sit in the trust boundary between block-group accounting and waiter wakeups, so refcount saturation, kCFI on dispatchers, and sanitize-on-free are required to stop ticket-wake UAF, dispatcher hijack, and ENOSPC-channel leaks.

## Open Questions

- Should Rookery model `BTRFS_RESERVE_FLUSH_EMERGENCY` as a separate code-path or as a fallback flag on the existing reserve_bytes path? Upstream uses a single branch inside reserve_bytes (REQ-29 step 11) but a typed-API variant could surface the safety implications more clearly.
- How should dynamic_reclaim threshold interact with zoned-mode `bg_reclaim_threshold = 75`? Upstream uses dynamic threshold only when `READ_ONCE(space_info.dynamic_reclaim)` is set; the zoned defaults are static.
- Sub-group accounting (`BTRFS_SUB_GROUP_TREELOG`, `_DATA_RELOC`): should counters roll up to parent or remain isolated for ENOSPC decisions? Current code calls `do_async_reclaim_*` per-subgroup independently.

## Out of Scope

- fs/btrfs/block-rsv.c (block_rsv counters and `btrfs_block_rsv_*`) — covered separately if expanded.
- fs/btrfs/extent-tree.c (extent allocation that consumes reserved bytes) — covered in `extent-tree.md` Tier-3.
- fs/btrfs/transaction.c (transaction lifecycle that flushes pinned + delayed) — covered in `transaction.md` Tier-3.
- fs/btrfs/delayed-ref.c (delayed-ref reservation consumers) — covered separately if expanded.
- fs/btrfs/zoned.c (RECLAIM_ZONES / RESET_ZONES helpers) — covered separately if expanded.
- fs/btrfs/relocation.c (relocation-driven reclaim) — covered in `relocation.md` Tier-3.
- fs/btrfs/free-space-cache.c (per-BG free-space tracking) — covered in `free-space-cache.md` Tier-3.
- Implementation code.
