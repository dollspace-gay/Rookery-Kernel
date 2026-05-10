# Tier-3: drivers/md/dm-mpath.c — multipath block target (active-passive / active-active path failover + per-priority-group path selection)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/dm-core.md
upstream-paths:
  - drivers/md/dm-mpath.c
  - drivers/md/dm-mpath.h
  - drivers/md/dm-path-selector.c
  - drivers/md/dm-path-selector.h
  - drivers/md/dm-uevent.c
-->

## Summary

dm-multipath is the device-mapper target that aggregates multiple paths to the same SAN/iSCSI block-device into one logical block-device — handles per-path failure detection + per-path failover + per-bio path-selection. Per-priority-group: pgs ordered by priority (e.g., active-passive: only first PG used; active-active: all PGs used in parallel). Per-PG path-selector plug-in (round-robin / queue-length / service-time) decides per-bio which path to dispatch. Critical for SAN deployments needing path-redundancy + load-balance. Userspace `multipathd` daemon manages path-state via uevents.

This Tier-3 covers `drivers/md/dm-mpath.c` (~2385 lines) + `dm-path-selector.c` (~138).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct multipath` | per-target state | `drivers::md::mpath::Multipath` |
| `struct priority_group` | per-PG state | `PriorityGroup` |
| `struct pgpath` | per-path state | `PgPath` |
| `struct path_selector` | per-PG selector instance | `PathSelector` |
| `struct path_selector_type` | per-selector vtable | `PathSelectorType` |
| `multipath_ctr(target, argc, argv)` | target-create | `Multipath::ctr` |
| `multipath_dtr(target)` | dtor | `Multipath::dtr` |
| `multipath_map_bio(target, bio)` | per-bio dispatch | `Multipath::map_bio` |
| `multipath_clone_and_map(...)` | per-bio path-selection + clone | `Multipath::clone_and_map` |
| `multipath_end_io(target, clone, error)` | per-bio completion | `Multipath::end_io` |
| `multipath_status(...)` | dmsetup status | `Multipath::status` |
| `multipath_message(...)` | DM_TARGET_MSG | `Multipath::message` |
| `multipath_iterate_devices(...)` | per-dev iter | `Multipath::iterate_devices` |
| `__pg_init_all_paths(m)` | per-PG path init | `Multipath::pg_init_all_paths` |
| `pg_init_all_paths(m)` | external hardware-handler init | `Multipath::pg_init_all_paths_extern` |
| `activate_or_offline_path(...)` | per-path state-change | `PgPath::activate_or_offline` |
| `__choose_pgpath(m, nr_bytes)` | per-bio path-selection | `Multipath::choose_pgpath` |
| `dm_register_path_selector(type)` (dm-path-selector.c) | per-selector register | `PathSelector::register` |
| `path_selector_create(name, ...)` | per-selector create | `PathSelector::create` |
| `dm_send_uevent(...)` (dm-uevent.c) | per-path uevent send | `DmUevent::send` |
| Selector types: `round-robin` / `queue-length` / `service-time` / `io-affinity` / `historical-service-time` | per-selector impls | `PathSelector::ROUND_ROBIN` / etc. |

## Compatibility contract

REQ-1: Per-target `multipath`:
- `ti` (back-ref).
- `priority_groups` (KVec<PriorityGroup>).
- `nr_priority_groups`.
- `current_pg` / `next_pg` (current PG for failover).
- `current_pgpath` (current path within PG).
- `flags` (MPATHF_*: QUEUE_IO, NO_FAIL, RETAIN_ATTACHED_HW_HANDLER, etc.).
- `nr_valid_paths` / `nr_active_paths` (counts).
- `pg_init_count` / `pg_init_required` (init coordination).
- `pg_init_in_progress` (atomic).
- `lock` (per-target spinlock).
- `hw_handler_name` (e.g., "alua" - SCSI ALUA).

REQ-2: Per-PG `priority_group`:
- `pgpaths` (KVec<PgPath>).
- `nr_pgpaths`.
- `bypassed` (skip during failover).
- `priority`.
- `ps` (PathSelector instance).

REQ-3: Per-path `pgpath`:
- `path` (DmPath struct: dev + state).
- `pg` (back-ref).
- `is_active` (bool).
- `fail_count`.
- `last_io_started_in_jiffies`.

REQ-4: Per-bio dispatch (`multipath_map_bio`):
1. m := target.private.
2. lock(m.lock).
3. pgpath := __choose_pgpath(m, bio.bi_iter.bi_size).
4. If !pgpath:
   - If MPATHF_QUEUE_IO: queue bio for retry; return DM_MAPIO_REQUEUE.
   - Else: bio_io_error.
5. unlock.
6. clone := bio_clone with bi_bdev = pgpath.path.dev.bdev.
7. submit_bio_noacct(clone).
8. Return DM_MAPIO_SUBMITTED.

REQ-5: Per-PG path-selection (`__choose_pgpath`):
1. If !current_pg: choose first non-bypassed PG.
2. ps := current_pg.ps.
3. pgpath := ps.type_.select_path(ps, &bytes).
4. If !pgpath: try next PG.
5. Return pgpath.

REQ-6: Per-bio completion (`multipath_end_io`):
1. If clone.bi_status & EIO: 
   - Mark pgpath failed.
   - dm_send_uevent(DM_UEVENT_PATH_FAILED).
   - If MPATHF_NO_FAIL: requeue bio.
   - Else: bio_io_error master_bio.
2. Else: bio_endio master_bio.

REQ-7: Path-selector types:
- round-robin: rotate through paths.
- queue-length: pick path with shortest queue.
- service-time: pick path with shortest avg service time.
- io-affinity: bind per-bio NUMA-affinity.
- historical-service-time: ML-based prediction.

REQ-8: Hardware-handler (per-array vendor):
- Hardware-handlers know per-vendor SAN-specific path-init protocol (e.g., "alua" for SCSI ALUA, "rdac" for LSI/NetApp).
- Per-target hw_handler_name configures.

REQ-9: Path-failure uevent:
- dm-uevent.c sends uevent on path failure / path activation / pg switch.
- Userspace multipathd subscribes; updates /etc/multipath.conf state.

REQ-10: Path-init coordination:
- Some HW-handlers require multi-step init (e.g., ALUA requires RTPG + STPG SCSI commands).
- pg_init_in_progress atomic gates concurrent inits.

REQ-11: Per-bio QUEUE_IO flag:
- MPATHF_QUEUE_IO: queue bios when no path available (don't fail immediately).
- Used during transient path failures.

REQ-12: Per-target features:
- `queue_if_no_path`: keep IO queued if all paths fail (vs immediate -EIO).
- `retain_attached_hw_handler`: keep HW-handler across path-changes.
- `pg_init_retries`: retry per-PG-init N times.
- `pg_init_delay_msecs`: delay between retries.

## Acceptance Criteria

- [ ] AC-1: dmsetup create test_mpath: 2 paths to same iSCSI LUN; bio dispatched via path-selector.
- [ ] AC-2: Path-failure: pull one cable; subsequent bio routes via second path; uevent fired.
- [ ] AC-3: Round-robin: 100 bios; per-path-IO approximately equal.
- [ ] AC-4: queue-length: bias toward path with shorter queue.
- [ ] AC-5: queue_if_no_path: all paths fail; bios queued; no -EIO.
- [ ] AC-6: PG failover: active PG goes down; subsequent bios routed via standby PG; pg_init runs.
- [ ] AC-7: ALUA hw-handler: per-vendor init protocol completes; subsequent IO succeeds.
- [ ] AC-8: 8-path stress: 100K bios across 8 paths; load-balanced; no hung path.
- [ ] AC-9: Path-restoration: pulled cable re-connected; multipathd uevent re-activates path.
- [ ] AC-10: dm-mpath xfstests pass.

## Architecture

`Multipath`:

```
struct Multipath {
  ti: KArc<DmTarget>,
  priority_groups: KVec<PriorityGroup>,
  nr_priority_groups: u32,
  current_pg: AtomicPtr<PriorityGroup>,
  next_pg: AtomicPtr<PriorityGroup>,
  current_pgpath: AtomicPtr<PgPath>,
  flags: u64,                                  // MPATHF_*
  nr_valid_paths: AtomicI32,
  nr_active_paths: AtomicI32,
  pg_init_count: AtomicI32,
  pg_init_required: AtomicI32,
  pg_init_in_progress: AtomicI32,
  pg_init_retries: u32,
  pg_init_delay_msecs: u32,
  pg_init_disabled: AtomicBool,
  lock: SpinLock<()>,
  queued_ios: BioList,                         // queued during path-failure
  hw_handler_name: KString,
  hw_handler_params: KString,
  ...
}

struct PriorityGroup {
  list: ListNode,
  m: KWeak<Multipath>,
  ps: KArc<PathSelector>,
  pgpaths: KVec<PgPath>,
  nr_pgpaths: u32,
  bypassed: AtomicBool,
}

struct PgPath {
  list: ListNode,
  path: KArc<DmPath>,
  pg: KWeak<PriorityGroup>,
  is_active: AtomicBool,
  fail_count: AtomicI32,
  last_io_started: u64,                         // jiffies
}

struct PathSelector {
  type_: KArc<PathSelectorType>,
  context: KBox<dyn PathSelectorContext>,
}

struct PathSelectorType {
  name: KStr,
  module: KModule,
  ctr: PathSelectorCtrFn,
  dtr: PathSelectorDtrFn,
  add_path: PathSelectorAddPathFn,
  fail_path: PathSelectorFailPathFn,
  reinstate_path: PathSelectorReinstatePathFn,
  select_path: PathSelectorSelectPathFn,
  start_io: PathSelectorStartIoFn,
  end_io: PathSelectorEndIoFn,
}
```

`Multipath::map_bio(target, bio)`:
1. m := target.private.
2. lock(m.lock).
3. pgpath := __choose_pgpath(m, bio.bi_iter.bi_size).
4. If !pgpath:
   - If m.flags & MPATHF_QUEUE_IO:
     - bio_list_add(&m.queued_ios, bio).
     - unlock.
     - return DM_MAPIO_SUBMITTED.
   - Else:
     - unlock.
     - bio_io_error(bio).
     - return DM_MAPIO_SUBMITTED.
5. unlock.
6. clone := bio_clone(bio).
7. clone.bi_bdev = pgpath.path.dev.bdev.
8. submit_bio_noacct(clone).
9. return DM_MAPIO_SUBMITTED.

`Multipath::__choose_pgpath(m, nr_bytes)`:
1. If !m.current_pg: select first non-bypassed PG.
2. ps := m.current_pg.ps.
3. pgpath := ps.type_.select_path(ps, &nr_bytes).
4. Return pgpath.

`Multipath::end_io(target, clone, &error)` flow:
1. If !error: bio_endio master_bio.
2. Else (error):
   - pgpath := lookup-by-clone.
   - mark pgpath.is_active = false.
   - dm_send_uevent(DM_UEVENT_PATH_FAILED).
   - If MPATHF_QUEUE_IO: requeue master_bio.
   - Else: bio_io_error master_bio.

`PathSelector::round_robin_select_path(ps, &nr_bytes)`:
1. ctx := ps.context as RoundRobin.
2. Loop:
   - candidate := ctx.paths[ctx.next_idx].
   - ctx.next_idx = (ctx.next_idx + 1) % ctx.path_count.
   - If candidate.is_active: return candidate.
3. Return NULL.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pgpath_idx_bounded` | OOB | per-PG.pgpaths indexed via select_path; bounded by nr_pgpaths. |
| `current_pg_validated` | INVARIANT | m.current_pg always points to valid PG or NULL. |
| `pg_init_count_no_underflow` | INVARIANT | pg_init_count ≥ 0; defense against double-decrement. |
| `bio_queued_no_uaf` | UAF | per-queued bio held under m.lock; defense against concurrent dispatch + free. |
| `path_state_atomic` | INVARIANT | path is_active transitions atomic. |

### Layer 2: TLA+

`drivers/md/multipath_failover.tla`:
- Per-path state ∈ {Active, Failed, Initializing, Bypassed}.
- Per-PG state ∈ {Selected, Standby, Bypassed}.
- Properties:
  - `safety_pg_init_serial` — only one PG-init in flight at a time.
  - `safety_failover_to_next_pg` — when current_pg fails, next_pg selected.
  - `liveness_active_path_eventually_chosen` — assuming any active path exists, select_path eventually returns one.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Multipath::map_bio` post: bio dispatched (path available) OR queued/error'd | `Multipath::map_bio` |
| `Multipath::end_io` post: per-error pgpath marked failed; uevent sent | `Multipath::end_io` |
| `Multipath::__choose_pgpath` post: returned pgpath active OR NULL | `Multipath::__choose_pgpath` |
| `PathSelector::*_select_path` post: returned candidate active per per-selector logic | per-selector |

### Layer 4: Verus/Creusot functional

`Per-bio: bio routed via active path; bio result equivalent to single-path direct IO` semantic equivalence: per-bio the (data, side-effect) match what a single-path direct submission would produce, modulo path-failover-retry on transient failure.

## Hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-multipath-specific reinforcement:

- **Per-target lock + atomic PG/path state** — defense against concurrent path-state-change with dispatch.
- **PG-init mutex via pg_init_in_progress atomic** — defense against concurrent init causing duplicate SCSI commands.
- **queue_if_no_path opt-in** — defense against silent IO failure on transient all-paths-down.
- **Per-path fail_count cap** — defense against transient-error flapping.
- **uevent rate-limit** — defense against multipathd flood on rapid path-flap.
- **HW-handler-name validated against registered handlers** — defense against attacker-supplied bogus name.
- **Per-PG bypassed flag honored at select** — defense against re-using known-bad PG.
- **dm_send_uevent under m.lock release** — defense against deadlock between uevent + dm-core.
- **Per-queued-bio refcount** — defense against double-dispatch on PG init recovery.
- **Per-path NUMA-affinity for io-affinity selector** — defense against pathological cross-NUMA cacheline ping-pong.
- **HW-handler init retry-bounded** — defense against unbounded init causing IO-stall.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- multipathd userspace daemon
- Per-vendor HW-handlers (covered in `drivers/md/dm-mpath-alua.md` future Tier-3)
- SCSI mid-layer (covered in `drivers/scsi/scsi-mid-core.md` Tier-3)
- iSCSI target (covered separately)
- Implementation code
