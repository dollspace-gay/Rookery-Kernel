# Tier-3: drivers/md/dm.c — device-mapper core (per-mapped-device target stacking + bio dispatch + ioctl mgmt)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/00-overview.md
upstream-paths:
  - drivers/md/dm.c
  - drivers/md/dm-core.h
  - drivers/md/dm-table.c
  - drivers/md/dm-ioctl.c
  - drivers/md/dm-target.c
  - drivers/md/dm.h
-->

## Summary

device-mapper (dm) is Linux's stackable block-device translation layer: a `mapped_device` accepts bio's, walks a per-md `dm_table` of `dm_target`s (each covering an LBA range), and dispatches each per-target type's map function (linear, striped, mirror, raid, crypt, thin-pool, snapshot, multipath, etc.). LVM2 + cryptsetup + thin-provisioning + RAID-on-block + iSCSI multipath all sit on top. dm-ioctl.c provides `/dev/mapper/control` with DM_DEV_CREATE / DM_TABLE_LOAD / DM_DEV_SUSPEND / DM_DEV_RESUME ioctls. Per-md two-table layout (active + inactive) lets userspace prepare new mapping then atomic-swap on resume.

This Tier-3 covers `dm.c` (~3830) + `dm-table.c` (~2231) + `dm-ioctl.c` (~2369).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mapped_device` | per-md state | `drivers::md::dm::MappedDevice` |
| `struct dm_table` | per-mapping table (active / inactive) | `DmTable` |
| `struct dm_target` | per-target descriptor | `DmTarget` |
| `struct dm_target_type` | per-type ops vtable (linear/striped/etc.) | `DmTargetType` |
| `dm_create(minor, &md)` (dm.c) | per-md alloc | `MappedDevice::create` |
| `dm_destroy(md)` | per-md teardown | `MappedDevice::destroy` |
| `dm_swap_table(md, table)` | atomic table swap (suspend/resume) | `MappedDevice::swap_table` |
| `dm_table_create(...)` (dm-table.c) | empty table alloc | `DmTable::create` |
| `dm_table_add_target(t, type, start, len, params)` | per-target add | `DmTable::add_target` |
| `dm_table_complete(t)` | freeze target list + sort | `DmTable::complete` |
| `dm_get_target(t, &target)` | lookup by sector | `DmTable::get_target` |
| `dm_register_target(type)` (dm-target.c) | per-type registration | `DmTarget::register_type` |
| `__map_bio(ti, bio)` (dm.c) | per-bio per-target dispatch | `MappedDevice::map_bio` |
| `dm_split_and_process_bio(...)` | per-target-boundary split | `MappedDevice::split_and_process_bio` |
| `dm_io(...)` (dm.c) | per-bio submission helper | `Dm::io` |
| `dm_get_md(...)` | per-bdev → md lookup | `MappedDevice::get_md` |
| `dm_request_queue(md)` | request queue accessor | `MappedDevice::request_queue` |
| `dm_suspend(md, flags)` (dm.c) | suspend IO | `MappedDevice::suspend` |
| `dm_resume(md)` | resume IO | `MappedDevice::resume` |
| `dm_ioctl(...)` (dm-ioctl.c) | /dev/mapper/control dispatcher | `DmIoctl::ioctl` |
| `dm_dev_create(...)` ioctl handler | DM_DEV_CREATE | `DmIoctl::dev_create` |
| `dm_dev_remove(...)` | DM_DEV_REMOVE | `DmIoctl::dev_remove` |
| `dm_table_load(...)` | DM_TABLE_LOAD | `DmIoctl::table_load` |
| `dm_dev_status(...)` | DM_DEV_STATUS | `DmIoctl::dev_status` |
| `dm_get_stripe_id(md)` (legacy) | per-md ID | `MappedDevice::get_id` |

## Compatibility contract

REQ-1: Per-md `mapped_device`:
- `block_device` (per-md /dev/dm-N gendisk).
- `lock` (rw_semaphore protecting tables).
- `map` (current active dm_table; RCU-pointer).
- `inactive` (loaded-but-not-active table).
- `wait` (waitqueue for suspend/resume sync).
- `pending_io` (in-flight bio counter).
- `frozen_sb` (frozen filesystem block).
- `flags` (DMF_BLOCK_IO_FOR_SUSPEND / DMF_SUSPENDED / DMF_NOFLUSH_SUSPENDING / etc.).

REQ-2: Per-table `dm_table`:
- `targets` (KVec<DmTarget>; sorted by start sector).
- `num_targets`.
- `num_allocated`.
- `highs` (per-target end-sector for sorted-binary-search).
- `mode` (FMODE_READ / _WRITE).
- `holders` (KVec<dm_dev*> of underlying block devices).
- `event_callback` (notify userspace on table-event).

REQ-3: Per-target `dm_target`:
- `table` (back-ref).
- `type` (target type vtable).
- `begin` (start sector in mapped_device).
- `len` (length in sectors).
- `private` (per-target instance state).
- `flush_supported` (bool).
- `discards_supported` (bool).
- `dax_supported` (bool).
- `error_buf` (per-target error string for ioctl).

REQ-4: Per-type `dm_target_type`:
- `name` (e.g., "linear", "striped", "crypt", "thin", "raid").
- `version` ([major, minor, patch]).
- `module` (kmod owner).
- `ctr(target, argc, argv)` (constructor).
- `dtr(target)` (destructor).
- `map(target, bio)` (per-bio remap; returns DM_MAPIO_REMAPPED / _SUBMITTED / _REQUEUE / _KILL).
- `end_io(target, bio, &error)` (per-bio post-IO).
- `presuspend(target)` / `postsuspend(target)` / `preresume(target)` / `resume(target)`.
- `status(target, status_type, ...)` (formatted status for ioctl).
- `message(target, argc, argv)` (DM_TARGET_MSG ioctl per-target).
- `iterate_devices(target, fn, data)` (walk underlying devs).
- `io_hints(target, &limits)` (queue limits).

REQ-5: Per-md bio submission flow:
1. submit_bio → per-md request-queue → dm_submit_bio (registered as queue.make_request_fn).
2. RCU-lock map.
3. Walk targets via binary-search on highs[]; find target covering bio.bi_iter.bi_sector.
4. If bio crosses target boundary: dm_split_and_process_bio splits into per-target sub-bios.
5. Per-sub-bio: type.map(target, sub_bio):
   - DM_MAPIO_REMAPPED: bio remapped to new bdev/sector; queue submit_bio_noacct.
   - DM_MAPIO_SUBMITTED: target submitted directly (e.g., dm-zero discards).
   - DM_MAPIO_REQUEUE: requeue.
   - DM_MAPIO_KILL: abort with error.
6. On completion: type.end_io(target, bio, &error) → bio_endio.
7. RCU-unlock map.

REQ-6: dm_table_load (ioctl):
1. Allocate dm_table.
2. For each target description in user-supplied buffer:
   - dm_table_add_target(t, type_name, start, len, params).
   - type := dm_get_target_type(type_name).
   - allocate dm_target with type.ctr(target, argc, argv).
3. dm_table_complete(t):
   - sort targets by start sector.
   - validate no gaps + no overlaps.
   - validate cumulative length matches md size.
4. md.inactive = t.

REQ-7: DM_DEV_SUSPEND ioctl:
1. Set DMF_BLOCK_IO_FOR_SUSPEND.
2. Wait for in-flight bio's to complete.
3. Optionally freeze underlying FS (for FSFREEZE flag).
4. Per-target presuspend + postsuspend hooks.
5. Set DMF_SUSPENDED.

REQ-8: DM_DEV_RESUME ioctl:
1. md.swap_table(md.inactive):
   - Acquire write-lock.
   - Per-target preresume hook.
   - rcu_assign_pointer(md.map, md.inactive).
   - md.inactive = NULL.
   - synchronize_rcu (wait readers).
2. Clear DMF_SUSPENDED.
3. Wake waiters; resume bio submission.
4. Per-target resume hook.

REQ-9: dm_get_target binary-search:
1. n := dm_table.num_targets.
2. Binary-search on highs[] for sector ∈ (highs[i-1], highs[i]].
3. Return targets[i].

REQ-10: Per-md ioctl path:
- /dev/mapper/control receives all DM_* ioctls.
- Per-ioctl: looks up md by name/uuid/devno; dispatches to handler.

REQ-11: Per-target type registration:
- Module init calls dm_register_target(&type).
- Module exit calls dm_unregister_target(&type).
- type-list protected by dm_target_lock.

REQ-12: Per-md request-based variant:
- For SCSI multipath: requests rather than bios are queued.
- Request-based dm has separate request-queue + per-request dm_rq_target_io.

REQ-13: Per-md DAX support:
- Each target may declare dax_supported.
- mapped-device DAX read/write goes via target.iov_iter operations.

## Acceptance Criteria

- [ ] AC-1: dmsetup create test_linear → linear target on /dev/sda; bio writes to test pass through to sda.
- [ ] AC-2: dmsetup suspend / resume: in-flight bios complete before suspend; new bios block; resume releases.
- [ ] AC-3: dm-crypt: encrypt LUKS volume; mount; read/write transparent.
- [ ] AC-4: dm-thin: thin-pool with 100MB metadata; 100GB data; thin-volume creation + write.
- [ ] AC-5: dm-mirror / dm-raid: raid1 with 2 disks; one fails; reads continue from healthy.
- [ ] AC-6: Multi-target table: linear:0-1G, striped:1-2G, error:2-3G; bios in each range routed correctly.
- [ ] AC-7: Discard/Trim: dm-thin propagates discard to underlying SSD.
- [ ] AC-8: Live table reload: dmsetup reload + suspend + resume swaps tables atomically.
- [ ] AC-9: ioctl ABI: dmsetup binary against current libdevmapper succeeds for all ops.
- [ ] AC-10: Linux Test Project dm test suite passes.

## Architecture

`MappedDevice`:

```
struct MappedDevice {
  bdev: KArc<BlockDevice>,
  disk: KArc<GenDisk>,
  ref_count: AtomicI32,
  io_lock: SpinLock<()>,
  map: AtomicPtr<DmTable>,                    // current active (RCU)
  inactive: Option<KArc<DmTable>>,
  flags: AtomicU64,                            // DMF_*
  pending_io: AtomicI32,
  wait: WaitQueueHead,
  frozen_sb: Option<KArc<SuperBlock>>,
  geom: Geometry,
  uuid: KStr,
  name: KStr,
  ...
}

struct DmTable {
  refcount: AtomicI32,
  targets: KVec<DmTarget>,
  highs: KVec<u64>,                            // sorted high-sector boundaries
  mode: u32,
  holders: KVec<KArc<DmDev>>,
  event_atomic: KAtomicPtr<DmEvent>,
  event_lock: SpinLock<()>,
}

struct DmTarget {
  table: KWeak<DmTable>,
  type_: KArc<DmTargetType>,
  begin: u64,
  len: u64,
  private: KBox<dyn DmTargetPrivate>,         // type-erased per-target state
  flush_supported: bool,
  discards_supported: bool,
  dax_supported: bool,
  error_buf: KString,
}
```

`MappedDevice::map_bio(md, bio)`:
1. rcu_read_lock(); table := rcu_dereference(md.map).
2. ti := DmTable::get_target(table, bio.bi_iter.bi_sector).
3. If bio crosses target.end:
   - sub_bios := dm_split_and_process_bio(bio, ti).
   - For each sub_bio: recurse.
4. Else:
   - ret := ti.type_.map(ti, bio).
   - Switch ret:
     - DM_MAPIO_REMAPPED: bio re-targeted; submit_bio_noacct(bio).
     - DM_MAPIO_SUBMITTED: target handled.
     - DM_MAPIO_REQUEUE: queue for retry.
     - DM_MAPIO_KILL: bio_io_error(bio).
5. rcu_read_unlock().

`MappedDevice::swap_table(md, new_table)`:
1. Down write md.io_lock.
2. old := rcu_dereference(md.map).
3. Per-target in old: presuspend if not done.
4. Per-target in new: preresume.
5. rcu_assign_pointer(md.map, new_table).
6. synchronize_rcu().
7. Per-target in new: resume.
8. Per-target in old: postsuspend.
9. drop old refcount.
10. Up write.

`DmIoctl::ioctl(file, cmd, arg)`:
1. Switch cmd:
   - DM_VERSION: return current dm version.
   - DM_DEV_CREATE: alloc + register md.
   - DM_DEV_REMOVE: unregister + free md.
   - DM_TABLE_LOAD: parse table; install as md.inactive.
   - DM_DEV_SUSPEND: suspend io.
   - DM_DEV_STATUS: per-target status format to user buf.
   - DM_TARGET_MSG: per-target message ioctl.
   - DM_TABLE_DEPS: enumerate underlying devs.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `target_lookup_no_oob` | OOB | per-table.targets indexed by binary-search-result; bounded by num_targets. |
| `bio_remap_within_target` | INVARIANT | per-bio post-map sector ∈ target.bdev range; defense against remap-to-OOB. |
| `rcu_protected_table_swap` | UAF | per-md.map updated via rcu_assign_pointer + synchronize_rcu; readers safe. |
| `target_holders_refcount` | INVARIANT | per-table holders ref-count balanced across create/destroy. |
| `cumulative_length_match` | INVARIANT | per-table sum of target lengths == md size; defense against partial coverage. |

### Layer 2: TLA+

`drivers/md/dm_table_swap.tla`:
- States: NoTable, ActiveOnly, ActiveAndInactive, Suspending, Suspended.
- Transitions per ioctl ops.
- Properties:
  - `safety_no_iio_during_swap` — table swap waits for in-flight to complete (via synchronize_rcu).
  - `safety_atomic_swap` — md.map points to either old or new entirely; never partial.
  - `liveness_resume_eventually_continues` — Suspended → Active eventually after resume.

`drivers/md/dm_target_dispatch.tla`:
- Per-bio per-target lookup + dispatch.
- Properties:
  - `safety_dispatch_within_target` — bio dispatched to ti only if ti.begin ≤ bio.sector < ti.begin + ti.len.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MappedDevice::map_bio` post: bio either remapped to underlying dev or marked submitted/error | `MappedDevice::map_bio` |
| `MappedDevice::swap_table` post: md.map points to new_table; old refcount dropped | `MappedDevice::swap_table` |
| `DmTable::get_target` post: returned ti.begin ≤ sector < ti.begin + ti.len | `DmTable::get_target` |
| `DmTable::complete` post: targets sorted by begin; cumulative-length matches md size | `DmTable::complete` |
| Per-target type.ctr called once per add; type.dtr called once per remove | invariants on DmTable lifecycle |

### Layer 4: Verus/Creusot functional

`Per-bio: bio submitted to md → (after target.map) bio reaches correct underlying block device + sector` semantic equivalence: per-bio per-target the resulting LBA on underlying bdev matches what target.type.map specifies for the source LBA.

## Hardening

(Inherits row-1 features from `drivers/md/00-overview.md` § Hardening.)

dm-core-specific reinforcement:

- **Per-table cumulative-length validation** — defense against partial-coverage causing split-bio reaching uncovered range.
- **Per-target overlap detection** — defense against ambiguous mapping.
- **RCU-protected table swap** — defense against in-flight bio reading half-installed table.
- **synchronize_rcu before drop-old-table** — defense against UAF on per-target private state.
- **Per-target type.ctr error handling** — defense against partial-target-init leaving inconsistent state.
- **Per-md /dev/mapper/control privileged** — defense against unauthorized table modification.
- **DM_DEV_SUSPEND wait-for-in-flight** — defense against bio active on table-being-swapped.
- **Per-target.private type-erased KBox** — defense against type-confusion across target types.
- **Per-md flags atomic** — defense against torn flag state during suspend/resume.
- **Per-target.error_buf bounded** — defense against unbounded error message size.
- **Per-target version checking** — defense against userspace passing args for newer target version.
- **Per-table holders ref-count balanced** — defense against underlying-bdev free while held.
- **Per-md DAX gating per-target** — defense against DAX read on non-DAX-capable target.

## Grsecurity/PaX-style Reinforcement

Rationale: dm-core is the dispatch layer for every device-mapper target — every bio entering `/dev/mapper/*` is routed via `dm_target.map`, `dm_target.map_rq`, and `dm_target.end_io`. A corrupted target vtable or unauthenticated ioctl yields arbitrary block-layer redirection and is the canonical foothold for tampering with dm-crypt, dm-verity, dm-integrity above. The reinforcement below restates baseline PaX/grsec coverage applied to `drivers/md/dm-core.c` plus dm-core-specific reinforcement.

Baseline (cross-ref `drivers/md/00-overview.md` § Hardening):
- **PAX_USERCOPY**: dm-ioctl `DM_TABLE_LOAD` / `DM_TABLE_STATUS` / `DM_DEV_STATUS` / `DM_TARGET_MSG` copy buffers (PAGE-sized) live in slab caches whitelisted for USERCOPY; size validated against `dm_ioctl.data_size`.
- **PAX_KERNEXEC**: `dm_target_type` table (per-target ctr/dtr/map/end_io/...) registered via `dm_register_target` and placed `__ro_after_init`; cannot be repointed after registration.
- **PAX_RANDKSTACK**: ioctl entry + bio dispatch entry randomise kernel stack.
- **PAX_REFCOUNT**: per-`mapped_device` refcount, per-`dm_table` refcount, per-target version saturating counters.
- **PAX_MEMORY_SANITIZE**: per-`mapped_device.uuid` + per-target-private-data zeroed on `dm_table_destroy`; defends UUID / per-target-key disclosure.
- **PAX_UDEREF**: every ioctl copy-from/to-user uses guarded accessors; `dm_ioctl.data` offsets bounds-checked.
- **PAX_RAP / kCFI**: per-target `ctr`/`dtr`/`map`/`map_rq`/`end_io`/`presuspend`/`postsuspend`/`preresume`/`resume`/`status`/`message`/`merge`/`busy`/`iterate_devices`/`io_hints`/`prepare_ioctl`/`dax_zero_page_range`/`dax_recovery_write`/`report_zones` are kCFI-tagged; mismatched signature → BUG().
- **GRKERNSEC_HIDESYM**: per-`mapped_device` addr, per-`dm_table` addr, per-target-private addr never rendered to dm-ioctl status; `%pK` only.
- **GRKERNSEC_DMESG**: dm-core warnings ratelimited; CAP_SYSLOG to read.

dm-core-specific reinforcement:
- **dm-target ops PAX_RAP** — `dm_target_type` kCFI fingerprint includes target name + version; loading a target module whose vtable doesn't match the registered fingerprint refuses registration. Defends against rogue OOT target shadowing an in-tree name (e.g. fake "crypt").
- **ioctl CAP_SYS_ADMIN strict** — every `DM_*` ioctl (LOAD / SUSPEND / RESUME / RENAME / REMOVE / TARGET_MSG / TABLE_CLEAR) requires CAP_SYS_ADMIN in `init_user_ns` (not the caller's userns); refuses unprivileged user-namespace owner access even if device is owned. Defends against userns escape via dm-table redirection.
- **status PAX_USERCOPY** — `dm_get_status` PAGE-sized buffer per target lives in a USERCOPY-whitelisted slab cache; `target.status` callback writes through bounded `result + maxlen` arguments.
- **`dm_table_complete` integrity check** — at table-load finalize, validates each target's stride doesn't overflow `sector_t`, each target.private != kernel-text-region, target.type in registered list, `dm_table.num_targets * dm_target_size < 16 MiB`.
- **Suspend/resume serialised under `dm_md.suspend_lock`** — already in row-2; extended with grsec audit log on suspend/resume to flag ransomware-style mass-suspend.
- **Per-md flags atomic** — already in row-2; reinforced with PAX_REFCOUNT semantics on `DMF_*` bits where they are reference-counted (e.g., DMF_FREEING).
- **`dm_get_target_type` module pin** — `try_module_get(target_type.module)` saturating; defends target-module-unload while table holds it.
- **`prepare_ioctl` passthrough refuses non-bdev-owner caps** — only forwards SCSI/NVMe ioctls when caller has CAP_SYS_RAWIO; defends ioctl tunneling through a dm target to gain raw-device access.
- **`message` arg-list bounded** — `dm_split_args` per-arg max 128 bytes, per-message max 16 args; defends parser-flood DoS.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-target type implementations (linear, crypt, thin, etc.; covered in `drivers/md/dm-<type>.md` future Tier-3s)
- Block layer core (covered in `block/00-overview.md`)
- gendisk + bdev (covered in `block/00-overview.md`)
- LVM2 userspace (userspace concern)
- Implementation code
