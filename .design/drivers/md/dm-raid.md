# Tier-3: drivers/md/dm-raid.c — bridge from device-mapper to md-raid (raid1/5/6/10/dm-raid LVM2 RAID-LV)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/dm-core.md
upstream-paths:
  - drivers/md/dm-raid.c
  - drivers/md/md.c (md-personality interface)
-->

## Summary

dm-raid bridges the device-mapper framework to md-raid personalities — userspace LVM2 creates RAID LVs via dm-raid (dmsetup type "raid"), which internally instantiates the appropriate md-raid personality (raid1, raid4, raid5, raid6, raid10) and manages it through dm-target ops. This lets LVM2 use modern md-raid backends (with bitmap, journal, write-mostly, etc.) while remaining a logical volume managed by dm-core. dm-raid handles per-disk sub-device naming, raid-event-handling, raid-superblock metadata, and resync-progress reporting. Used by all modern LVM2 RAID configurations.

This Tier-3 covers `drivers/md/dm-raid.c` (~4176 lines).

## Upstream references

| Upstream symbol | Purpose | Rookever owner |
|---|---|---|
| `struct raid_set` | per-target raid config | `drivers::md::raid::RaidSet` |
| `struct raid_dev` | per-disk descriptor | `RaidDev` |
| `struct raid_type` | per-personality vtable | `RaidType` |
| `raid_ctr(target, argc, argv)` | target-create | `Raid::ctr` |
| `raid_dtr(target)` | dtor | `Raid::dtr` |
| `raid_map(target, bio)` | per-bio dispatch via md_handle_request | `Raid::map_bio` |
| `raid_status(target, type, ...)` | dmsetup status | `Raid::status` |
| `raid_message(target, argc, argv)` | DM_TARGET_MSG | `Raid::message` |
| `raid_iterate_devices(target, fn, data)` | per-dev iter | `Raid::iterate_devices` |
| `raid_io_hints(target, &limits)` | queue limits | `Raid::io_hints` |
| `raid_postsuspend(target)` / `_preresume(target)` | suspend/resume hooks | `Raid::postsuspend` / `_preresume` |
| `raid_resume(target)` | resume | `Raid::resume` |
| `raid_check_takeover(rs)` | takeover validation (raid1 → raid5 etc.) | `Raid::check_takeover` |
| `attempt_takeover(rs, level)` | takeover dispatch | `Raid::attempt_takeover` |
| `setup_md_personality(rs, level)` | per-level personality bind | `Raid::setup_md_personality` |
| `analyse_superblocks(target, rs)` | per-disk superblock parse | `Raid::analyse_superblocks` |
| `super_validate(rs, &sb)` | superblock validation | `Raid::super_validate` |
| `_raid_setup_md_disks(rs)` | md_dev binding | `Raid::setup_md_disks` |

## Compatibility contract

REQ-1: Per-target `raid_set`:
- `ti` (back-ref).
- `mddev` (md_dev embedded; bridged to md-raid).
- `raid_type` (KArc<RaidType>).
- `bitmap_loaded` (bool).
- `dev` (KVec<RaidDev>; per-disk).
- `chunk_size`.
- `level` (raid level: 1, 4, 5, 6, 10).
- `region_size` (bitmap region size).
- `flags` (per-target capability flags).
- `recovery_flags` (recovery state).

REQ-2: Per-disk `raid_dev`:
- `meta_dev` (DmDev for superblock + bitmap).
- `data_dev` (DmDev for actual data).
- `data_offset` (sector offset on data_dev).
- `meta_offset` (sector offset on meta_dev).
- `rdev` (md_rdev structure for md-raid layer).

REQ-3: dmsetup raid args:
- `dmsetup create raid_lv --table "0 N raid <level> <#args> <chunk_size> [bitmap=<size>] [...] <#disks> [<meta_dev> <data_dev>]+"`.

REQ-4: Per-target ctr flow:
1. Parse args; identify level + chunk_size + per-disk metas.
2. raid_type := lookup per-level vtable (raid1 / raid4 / raid5 / raid6 / raid10).
3. Allocate raid_set + raid_dev[].
4. analyse_superblocks: read per-disk md-superblock; validate consistency.
5. setup_md_personality: bind to md_personality; mddev_run.
6. Set target.flush_supported / discards_supported / etc.

REQ-5: Per-bio dispatch (`raid_map`):
1. rs := target.private.
2. bio.bi_iter.bi_sector += rs.dev[0].data_offset.
3. md_handle_request(&rs.mddev, bio) — dispatches to md-raid personality.
4. Return DM_MAPIO_SUBMITTED.

REQ-6: Per-target status:
- "raid <level> <state> <recovery_progress> <component_states>".
- state: idle / recovery / resync / etc.
- per-disk state: A=active, F=faulted, etc.

REQ-7: Per-target dmsetup messages:
- "fail <dev>": mark per-disk faulty.
- "remove <dev>": remove per-disk.
- "rebuild <dev>": trigger rebuild.
- "frozen": pause resync.
- "raid_resync": force resync.
- "writebehind <N>": set writebehind for write-mostly disks.

REQ-8: Per-target features:
- bitmap (intent log).
- journal device (for raid5/6 write-hole-defense).
- write-mostly (per-disk no-read).
- max_recovery_rate / min_recovery_rate (sysfs tunables).

REQ-9: Per-disk superblock:
- md_superblock_v1.0 / v1.1 / v1.2 (per layout variant).
- Contains: array UUID, role, generation, recovery offset, etc.
- Per-disk superblock validated at ctr.

REQ-10: Takeover:
- raid1 → raid5: convert mirror to parity-stripe.
- raid5 → raid6: add Q-parity.
- Per-takeover: temporary state during conversion.

REQ-11: Per-target md_dev integration:
- mddev embedded in raid_set; md-raid personality manipulates.
- md_personality.run / make_request / status / etc. dispatched via mddev.

REQ-12: Per-target sysfs integration:
- /sys/dm-N/md/* exposes md-raid sysfs attributes.
- mdadm-compatible monitoring.

## Acceptance Criteria

- [ ] AC-1: dmsetup create raid1: 2-disk mirror via dm-raid; bios writes to both.
- [ ] AC-2: dmsetup create raid5: 3-disk parity stripe; reads/writes correct.
- [ ] AC-3: Disk-fail: mark fail via dmsetup message; subsequent IO continues degraded.
- [ ] AC-4: Rebuild: replace faulted disk; recovery progress visible via status.
- [ ] AC-5: Takeover raid1 → raid5: data preserved; new parity disk added.
- [ ] AC-6: Bitmap: bitmap-tracked dirty regions; fast resync after power-loss.
- [ ] AC-7: Journal: raid5 with journal device; write-hole-defended on power-loss.
- [ ] AC-8: LVM2 mirrored LV: lvcreate --mirrors 1 vg/lv; uses dm-raid backend.
- [ ] AC-9: dmsetup status shows per-disk state + recovery-progress.
- [ ] AC-10: dm-raid xfstests pass.

## Architecture

`RaidSet`:

```
struct RaidSet {
  ti: KArc<DmTarget>,
  mddev: KBox<MdDev>,
  raid_type: KArc<RaidType>,
  bitmap_loaded: bool,
  dev: KVec<RaidDev>,
  chunk_size: u32,
  level: u8,
  region_size: u32,
  flags: u64,
  recovery_flags: u64,
  callbacks: KArc<MdSyncCallback>,
  ...
}

struct RaidDev {
  meta_dev: KArc<DmDev>,
  data_dev: KArc<DmDev>,
  data_offset: u64,
  meta_offset: u64,
  rdev: KArc<MdRdev>,
}

struct RaidType {
  name: KStr,
  level: u8,
  parity_devs: u32,
  algorithm: u32,
  ...
}
```

`Raid::ctr(target, argc, argv)` flow:
1. Parse argv to extract: level + chunk_size + bitmap + #disks + per-disk pairs.
2. lookup raid_type by level+algorithm.
3. Allocate rs := RaidSet.
4. For each (meta_dev, data_dev) pair:
   - Allocate raid_dev.
   - dm_get_device(meta_dev) + dm_get_device(data_dev).
   - Set rdev fields per md_rdev structure.
5. analyse_superblocks(rs):
   - Per-disk: read superblock; validate UUID + role + generation.
6. setup_md_personality(rs, level):
   - mddev.pers = lookup_personality(level).
   - mddev.pers->run(&mddev).
7. target.private = rs.

`Raid::map_bio(target, bio)`:
1. rs := target.private.
2. mddev := &rs.mddev.
3. md_handle_request(mddev, bio).
4. return DM_MAPIO_SUBMITTED.

`Raid::message(target, argc, argv)`:
1. Switch argv[0]:
   - "fail": mark per-disk faulty; md_error.
   - "remove": detach per-disk.
   - "rebuild": md_check_recovery.
   - "frozen": set MD_RECOVERY_FROZEN.
   - "writebehind": adjust per-disk write-mostly.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `raid_dev_count_match` | INVARIANT | rs.dev.len() == raid_type.required_disks; defense against partial config. |
| `chunk_size_power_of_2` | INVARIANT | chunk_size power-of-2; defense against modulo-overhead. |
| `superblock_validation_complete` | INVARIANT | analyse_superblocks called before mddev_run. |
| `md_dev_lifecycle_paired` | UAF | mddev_run / mddev_destroy paired across ctr/dtr. |
| `data_offset_within_dev` | INVARIANT | per-RaidDev.data_offset + array_size ≤ data_dev capacity. |

### Layer 2: TLA+

`drivers/md/dm_raid_lifecycle.tla`:
- Per-target state ∈ {Created, Suspended, Active, Recovering, Failed}.
- Per-disk state ∈ {InSync, Faulty, Spare, Recovering}.
- Properties:
  - `safety_personality_bound_after_ctr` — Active state implies md-personality.run completed.
  - `safety_recovery_after_disk_fail` — Faulty disk eventually triggers recovery if spare available.
  - `liveness_pending_eventually_in_sync` — Recovering → InSync assuming recovery completes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Raid::ctr` post: rs.mddev configured; per-disk md_rdev populated | `Raid::ctr` |
| `Raid::map_bio` post: bio dispatched to md_handle_request | `Raid::map_bio` |
| `Raid::dtr` post: mddev cleanly stopped; rs freed | `Raid::dtr` |
| Per-target lifecycle: ctr → resume → suspend → dtr ordered | invariants on lifecycle ops |

### Layer 4: Verus/Creusot functional

`Per-bio: dm-raid passes through to md-raid personality preserving level-specific semantics` semantic equivalence: per-bio result equivalent to using md-raid directly via /dev/mdN.

## Hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-raid-specific reinforcement:

- **Per-target chunk_size validated power-of-2** — defense against modulo-overhead.
- **Per-disk superblock validation at ctr** — defense against malformed disk-array.
- **Per-target md-personality lifecycle** — defense against md_dev leak on failed ctr.
- **Per-disk data_offset bounds** — defense against OOB device-write.
- **bitmap loaded if requested** — defense against silent bitmap-disable losing fast-resync.
- **journal device validated** — defense against missing journal causing write-hole on raid5/6.
- **Per-recovery-rate sysfs gating** — defense against unauthorized recovery-rate change.
- **Per-disk fail/remove ordered** — defense against parallel disk-state-change races.
- **Takeover progress journaled** — defense against crash-mid-takeover leaving inconsistent state.
- **Per-target reload + suspend/resume coordinated** — defense against concurrent table-swap with active recovery.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- md-raid1 / raid5 / raid10 personalities (covered in `raid1.md` / `raid5.md` / `raid10.md` Tier-3s)
- LVM2 userspace
- mdadm CLI
- Implementation code
