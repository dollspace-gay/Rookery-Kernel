# Tier-3: fs/btrfs/transaction.c — Btrfs transaction subsystem (atomic CoW commit)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/super.md
upstream-paths:
  - fs/btrfs/transaction.c (~2754 lines)
  - fs/btrfs/transaction.h
  - include/uapi/linux/btrfs.h (BTRFS_IOC_START_SYNC, BTRFS_IOC_WAIT_SYNC)
-->

## Summary

Btrfs uses **CoW transactions** to provide atomic, crash-consistent updates across multiple trees. Per-FS has a single in-progress `btrfs_transaction` (per-fs_info.running_transaction). Per-mutation (write, snapshot, balance, ...) joins the transaction via `btrfs_start_transaction` (returns `btrfs_trans_handle`); commit phase: pre-commit hooks, run delayed refs, run delayed inodes, write all dirty COW'd nodes, write super(s), barrier, advance generation. Per-commit serialized via state machine `TRANS_STATE_RUNNING → BLOCKED → COMMIT_DOING → COMMIT_DONE`. Critical for: btrfs durability, snapshot atomicity, balance/relocation consistency.

This Tier-3 covers `fs/btrfs/transaction.c` (~2754 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct btrfs_transaction` | per-txn state | `BtrfsTrans` |
| `struct btrfs_trans_handle` | per-handle | `BtrfsTransHandle` |
| `btrfs_start_transaction()` | per-acquire-handle | `Btrfs::start_transaction` |
| `btrfs_attach_transaction()` | per-attach-no-start | `Btrfs::attach_transaction` |
| `btrfs_join_transaction()` | per-join-existing-or-new | `Btrfs::join_transaction` |
| `btrfs_commit_transaction()` | per-commit | `Btrfs::commit_transaction` |
| `btrfs_end_transaction()` | per-release-handle | `Btrfs::end_transaction` |
| `btrfs_abort_transaction()` | per-abort | `Btrfs::abort_transaction` |
| `start_transaction()` (internal) | per-alloc | `Btrfs::start_transaction_inner` |
| `wait_current_trans()` | per-wait | `Btrfs::wait_current_trans` |
| `commit_cowonly_roots()` | per-commit roots | `Btrfs::commit_cowonly_roots` |
| `commit_fs_roots()` | per-fs root commit | `Btrfs::commit_fs_roots` |
| `update_super_roots()` | per-write-back super.root tree-bytenrs | `Btrfs::update_super_roots` |
| `btrfs_transaction_blocked()` | per-state-check | `BtrfsTrans::blocked` |
| `btrfs_should_end_transaction()` | per-end-hint | `Btrfs::should_end_transaction` |
| `transaction_kthread()` | per-fs kthread | `Btrfs::transaction_kthread` |
| `cleanup_transaction()` | per-abort cleanup | `Btrfs::cleanup_transaction` |

## Compatibility contract

REQ-1: struct btrfs_transaction:
- transid: u64 generation.
- num_writers: counter of active handles.
- num_extwriters: extended writers (e.g., relocation).
- blocked: # blocked-on-commit.
- state: TRANS_STATE_*.
- delayed_refs: btrfs_delayed_ref_root.
- pending_snapshots: list.
- pending_chunks: list.
- pending_ordered: list.
- dirty_bgs: list of dirty block-groups.
- io_bgs: I/O-pending block-groups.
- aborted: errno (0 = ok).
- start_time: ktime.

REQ-2: TRANS_STATE_*:
- RUNNING (0): accepting joins.
- COMMIT_PREP (1): preparing commit.
- COMMIT_START (2): blocking new joins.
- COMMIT_DOING (3): actively committing.
- UNBLOCKED (4): readers can proceed.
- SUPER_COMMITTED (5): super written.
- COMPLETED (6): fully done.

REQ-3: struct btrfs_trans_handle:
- transid: u64 of joined trans.
- bytes_reserved: bytes reserved from space-info.
- chunk_bytes_reserved: chunk reserve.
- delayed_ref_updates: count of pending updates this handle issued.
- block_rsv: bound block-reservation.
- type: TRANS_START / TRANS_JOIN / TRANS_USERSPACE / TRANS_ATTACH / TRANS_JOIN_NOLOCK / TRANS_JOIN_NOSTART.
- root: per-root context.
- aborted: errno.
- removing_chunk: bool.
- can_flush_pending_bgs: bool.

REQ-4: btrfs_start_transaction(root, num_items):
- Per-num_items: how many tree-items will be modified (used to reserve block-rsv).
- block_rsv_amount = btrfs_calc_metadata_size(fs_info, num_items).
- Reserve block-rsv.
- Wait for non-blocked transaction.
- num_writers++.
- Return trans_handle.

REQ-5: btrfs_join_transaction(root):
- Equivalent to start_transaction(root, 0) — no item reservation.
- Used when piggy-backing on existing trans.

REQ-6: btrfs_attach_transaction(root):
- Get reference to current trans without joining as writer.
- Used by sync(2) / btrfs-progs to wait for current trans.

REQ-7: btrfs_end_transaction(trans):
- num_writers--.
- if num_writers == 0 ∧ should_commit: kick commit-kthread.

REQ-8: btrfs_commit_transaction(trans):
- /* State transitions */
- TRANS_STATE_RUNNING → COMMIT_PREP (no new flush jobs).
- Run all pending hooks (run_delayed_refs to drain queue).
- → COMMIT_START (block new TRANS_START joins; TRANS_JOIN_NOLOCK still allowed).
- Wait num_writers ≤ extwriters → drain.
- → COMMIT_DOING.
- /* Iterate per-root dirty inodes */
- commit_fs_roots.
- /* Write back COW-only roots: extent-tree, dev-tree, csum-tree, fs-tree-roots subset */
- commit_cowonly_roots.
- /* Update super blocks */
- update_super_roots(fs_info).
- write_all_supers.
- → SUPER_COMMITTED.
- /* Cleanup pending snapshots, balance, etc. */
- → COMPLETED.

REQ-9: btrfs_abort_transaction(trans, errno):
- Set fs_info.fs_state |= BTRFS_FS_STATE_ERROR.
- trans.aborted = errno; current_trans.aborted = errno.
- Force RO mount.

REQ-10: Per-pre-commit hook ordering:
- run_delayed_refs (drain queue).
- create_pending_block_groups.
- run_delayed_iputs (free deleted-inodes' subvolumes).
- run_delayed_items (delayed-inode / delayed-dir-index).

REQ-11: Per-snapshot creation:
- ioctl(BTRFS_IOC_SNAP_CREATE_V2).
- Add to trans.pending_snapshots.
- Per-commit: create snapshot subvol-tree as CoW of source.

REQ-12: Per-fs-root commit:
- Each fs-root (subvolume) has its own tree.
- Per-commit: write back dirty fs-tree nodes.
- Update root-item to point to new tree-root bytenr.

REQ-13: Per-cowonly tree:
- extent-tree, dev-tree, csum-tree, chunk-tree.
- Always CoW'd per-write; never split-shared.

REQ-14: Per-write_all_supers:
- Write per-device super-block(s).
- Barrier (REQ_PREFLUSH | REQ_FUA).
- Crash before this: prior generation visible.
- Crash after: new generation visible.

REQ-15: Per-transaction_kthread:
- Wakes per-fs_info.commit_interval (default 30s).
- If running_trans elapsed: btrfs_commit_transaction.

REQ-16: Per-fsync (fast-commit-style):
- per-write log-tree (separate path from full transaction).
- Subset commit.

REQ-17: Per-num_items reserve:
- 1 item ≈ nodesize-worth of bytes.
- Reserved up-front to ensure commit can proceed without ENOSPC mid-flight.

REQ-18: Per-error: TRANS_ABORTED → fs RO + WARN_ON.

## Acceptance Criteria

- [ ] AC-1: btrfs_start_transaction(root, 5): trans_handle returned; num_writers++; bytes_reserved = 5 * nodesize.
- [ ] AC-2: btrfs_join_transaction: returns trans_handle for current trans.
- [ ] AC-3: btrfs_end_transaction: num_writers-- ; if 0 + should_commit: commit kicked.
- [ ] AC-4: btrfs_commit_transaction: state advances RUNNING → ... → COMPLETED.
- [ ] AC-5: Per-pending snapshot in trans: snapshot tree created on commit.
- [ ] AC-6: Per-cowonly trees written; super updated.
- [ ] AC-7: Per-write_all_supers PREFLUSH+FUA barriers.
- [ ] AC-8: btrfs_abort_transaction(trans, -EIO): fs-state ERROR; subsequent ops -EROFS.
- [ ] AC-9: transaction_kthread fires after commit_interval; commits if running_trans elapsed.
- [ ] AC-10: ioctl BTRFS_IOC_START_SYNC: returns transid; BTRFS_IOC_WAIT_SYNC waits for super-commit.
- [ ] AC-11: Crash recovery: log-tree replay or last-completed-trans visible.
- [ ] AC-12: Per-num_items insufficient: -ENOSPC at start_transaction (ahead-of-time).

## Architecture

```
struct BtrfsTrans {
  use_count: AtomicI32,
  transid: u64,
  num_writers: AtomicI32,
  num_extwriters: AtomicI32,
  state: AtomicU8,                  // TRANS_STATE_*
  blocked: AtomicI32,
  delayed_refs: BtrfsDelayedRefRoot,
  pending_snapshots: List<BtrfsPendingSnapshot>,
  dirty_bgs: List<BlockGroup>,
  io_bgs: List<BlockGroup>,
  aborted: AtomicI32,
  start_time: KTime,
  list: ListLink,                   // fs_info.trans_list
}

struct BtrfsTransHandle {
  transid: u64,
  bytes_reserved: u64,
  chunk_bytes_reserved: u64,
  type: u8,
  root: *BtrfsRoot,
  block_rsv: *BtrfsBlockRsv,
  delayed_ref_updates: u64,
  aborted: i32,
  removing_chunk: bool,
}
```

`Btrfs::start_transaction(root, num_items, type) -> Result<*TransHandle>`:
1. fs_info = root.fs_info.
2. /* Wait for non-blocked trans */
3. retry:
4. cur = fs_info.running_transaction.
5. if cur ∧ cur.state >= COMMIT_START ∧ type == TRANS_START:
   - wait_event(cur.commit_wait, cur.state >= UNBLOCKED).
   - goto retry.
6. /* Reserve metadata */
7. block_rsv = btrfs_get_block_rsv(fs_info, type).
8. err = btrfs_block_rsv_add(root, block_rsv, num_items * nodesize, FLUSH_LIMIT).
9. if err: return ERR_PTR(err).
10. /* Allocate handle */
11. trans = kmem_cache_zalloc(btrfs_trans_handle_cachep).
12. trans.transid = cur.transid; trans.bytes_reserved = num_items * nodesize.
13. trans.type = type; trans.root = root; trans.block_rsv = block_rsv.
14. atomic_inc(&cur.num_writers).
15. return trans.

`Btrfs::commit_transaction(trans) -> Result<()>`:
1. cur = trans_to_btrfs_trans(trans).
2. /* Wait if any other commit in progress */
3. wait_event(cur.commit_wait, cur.state == TRANS_STATE_RUNNING).
4. /* RUNNING → COMMIT_PREP */
5. cur.state = COMMIT_PREP.
6. /* Run delayed refs */
7. btrfs_run_delayed_refs(trans, U64_MAX).
8. /* Drain to COMMIT_START: block new starts */
9. cur.state = COMMIT_START.
10. wait_event(cur.writer_wait, atomic_read(&cur.num_writers) == 1).
11. /* COMMIT_DOING */
12. cur.state = COMMIT_DOING.
13. /* Pre-commit hooks */
14. btrfs_run_delayed_items(trans).
15. btrfs_run_delayed_iputs(fs_info).
16. /* Commit fs_roots */
17. ret = commit_fs_roots(trans).
18. /* Commit cowonly roots */
19. ret = commit_cowonly_roots(trans).
20. /* Pending snapshots created */
21. process_pending_snapshots(trans).
22. /* Update super roots */
23. update_super_roots(fs_info).
24. /* Write all supers */
25. ret = write_all_supers(fs_info, 0).
26. cur.state = SUPER_COMMITTED.
27. /* Cleanup */
28. btrfs_finish_extent_commit(trans).
29. cur.state = COMPLETED.
30. wake_up(&fs_info.transaction_wait).

`Btrfs::end_transaction(trans) -> Result<()>`:
1. cur = trans_to_btrfs_trans(trans).
2. /* Release block reserve */
3. btrfs_block_rsv_release(fs_info, trans.block_rsv, trans.bytes_reserved, NULL).
4. atomic_dec(&cur.num_writers).
5. /* Should-commit hint */
6. if Btrfs::should_end_transaction(trans):
   - kick commit-kthread.
7. kmem_cache_free(btrfs_trans_handle_cachep, trans).

`Btrfs::abort_transaction(trans, err)`:
1. cur = trans_to_btrfs_trans(trans).
2. if !cur: return.
3. WRITE_ONCE(cur.aborted, err).
4. WRITE_ONCE(trans.aborted, err).
5. fs_info.fs_state |= BTRFS_FS_STATE_ERROR.
6. /* RO mount */
7. btrfs_handle_fs_error(fs_info, err, "transaction aborted").

`Btrfs::transaction_kthread(arg)`:
1. fs_info = arg.
2. while !kthread_should_stop():
   - wait_for_pulse(commit_interval ms).
   - cur = fs_info.running_transaction.
   - if !cur: continue.
   - if (jiffies - cur.start_time_jiffies) ≥ commit_interval:
     - trans = btrfs_attach_transaction(fs_info.tree_root).
     - btrfs_commit_transaction(trans).

`Btrfs::should_end_transaction(trans) -> bool`:
1. /* Heuristic: many delayed-ref updates ⟹ commit soon */
2. fs_info = trans.root.fs_info.
3. if test_bit(BTRFS_FS_NEED_ASYNC_COMMIT, &fs_info.flags): return true.
4. if trans.delayed_ref_updates > batch_size: return true.
5. return false.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `state_monotonic` | INVARIANT | per-trans state advances only forward (RUNNING → ... → COMPLETED). |
| `num_writers_nonneg` | INVARIANT | per-trans: num_writers ≥ 0. |
| `block_rsv_balanced` | INVARIANT | per-handle: bytes_reserved acquired then released. |
| `cur_trans_unique_per_fs` | INVARIANT | per-fs_info: at most 1 running_transaction. |
| `commit_only_in_dispatch_state` | INVARIANT | per-commit: state ∈ {RUNNING, COMMIT_PREP entering}. |
| `aborted_implies_fs_state_error` | INVARIANT | per-trans.aborted != 0 ⟹ fs_state has ERROR. |

### Layer 2: TLA+

`fs/btrfs/transaction.tla`:
- Per-transaction state-machine + per-handle join/end + per-commit.
- Per-concurrent-handles + per-commit-blocking.
- Properties:
  - `safety_no_join_after_commit_start` — per-state >= COMMIT_START: no new TRANS_START joins.
  - `safety_no_writers_during_commit_doing` — per-COMMIT_DOING: num_writers == 1 (commit-issuer).
  - `safety_super_committed_implies_durable` — per-SUPER_COMMITTED: no rollback.
  - `liveness_per_running_eventually_commits` — per-running-trans: after ≤ commit_interval, COMPLETED.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Btrfs::start_transaction` post: handle.transid == cur.transid; num_writers++ | `Btrfs::start_transaction` |
| `Btrfs::commit_transaction` post: state == COMPLETED | `Btrfs::commit_transaction` |
| `Btrfs::end_transaction` post: num_writers--; bytes returned to block_rsv | `Btrfs::end_transaction` |
| `Btrfs::abort_transaction` post: fs_state has ERROR; trans.aborted set | `Btrfs::abort_transaction` |
| `commit_cowonly_roots` post: cowonly trees written | `commit_cowonly_roots` |
| `update_super_roots` post: fs_info.super_copy.{root, chunk_root, dev_root, ...} updated | `update_super_roots` |

### Layer 4: Verus/Creusot functional

`Per-transaction CoW commit: handles join → mutations → run_delayed_refs → commit_fs_roots → commit_cowonly_roots → write_all_supers → atomic generation-advance` semantic equivalence: per-Documentation/filesystems/btrfs.rst (transactions section).

## Hardening

(Inherits row-1 features from `fs/btrfs/super.md` § Hardening.)

Transaction reinforcement:

- **Per-block-rsv pre-reserved up-front** — defense against per-mid-commit ENOSPC.
- **Per-state monotonic advance** — defense against per-rollback-bug.
- **Per-aborted flag bound to fs_state ERROR** — defense against per-stale-trans-after-abort.
- **Per-trans handle lifecycle: start → end paired** — defense against per-leak.
- **Per-write_all_supers PREFLUSH+FUA** — defense against per-crash-mid-super.
- **Per-running_transaction unique per-fs** — defense against per-double-trans.
- **Per-num_writers atomic counters** — defense against per-race counter.
- **Per-commit kthread interval-bounded** — defense against per-delay-too-long.
- **Per-pending-snapshot list-locked** — defense against per-concurrent-snap.
- **Per-trans.delayed_refs drained pre-commit** — defense against per-orphan ref.
- **Per-CAP_SYS_ADMIN for ioctl(START_SYNC, WAIT_SYNC)** — defense against unprivileged commit-trigger.
- **Per-fs_state ERROR forces RO** — defense against per-corrupted-write-after-abort.
- **Per-block_rsv release on end** — defense against per-rsv-leak.

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
- **Transaction PAX_REFCOUNT** — saturating refcount on `btrfs_trans_handle` and `running_transaction`, blocking ref-overflow attacks across `btrfs_join_transaction` / `btrfs_end_transaction`.
- **Super-block write barrier integrity** — `write_all_supers` with PREFLUSH+FUA flagged as ROOKERY_BARRIER region; kCFI on the I/O completion callbacks so post-commit super writeout cannot be redirected.
- **PAX_MEMORY_SANITIZE on trans_handle dealloc** — `trans_handle` carries `block_rsv` and pending-snapshot pointers; on free the struct is zeroed before slab return.
- **PAX_USERCOPY on ioctl(START_SYNC / WAIT_SYNC)** — bounded copy from user-space transid arg.
- **GRKERNSEC_HIDESYM on transaction-abort dump** — `BTRFS_FS_ERROR` diagnostic suppresses kernel pointer disclosure.
- **PAX_REFCOUNT on num_writers** — atomic counter saturates rather than wraps on writer-storm.
- **kCFI on commit kthread entry** — `transaction_kthread` only entered through signed function pointer.

Per-doc rationale: btrfs transactions arbitrate cross-tree atomicity via `running_transaction`, writer/blocked counters, and a state machine that must monotonically advance to RUNNING → COMMIT_PREP → UNBLOCKED → SUPER_COMMITTED → COMPLETED; refcount saturation, sanitize-on-free, and kCFI on the commit dispatcher and IO completion paths protect the boundary between in-memory commit state and on-disk super writeout.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/btrfs/super.c (covered in `super.md` Tier-3)
- fs/btrfs/extent-tree.c (covered in `extent-tree.md` Tier-3)
- fs/btrfs/disk-io.c (covered separately if expanded)
- fs/btrfs/tree-log.c log-tree (covered separately if expanded)
- fs/btrfs/space-info.c (covered separately if expanded)
- fs/btrfs/block-rsv.c (covered separately if expanded)
- Implementation code
