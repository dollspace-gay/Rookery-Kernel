# Tier-3: sound/core/init.c — ALSA card init / register / disconnect / free

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: sound/00-overview.md
upstream-paths:
  - sound/core/init.c (~1176 lines)
  - sound/core/device.c
  - include/sound/core.h
  - include/uapi/sound/asound.h
-->

## Summary

ALSA core lifecycle is anchored in `struct snd_card`. A driver allocates a card via `snd_card_new()`, attaches per-component `struct snd_device` instances (control, PCM, raw-MIDI, timer, sequencer, hwdep, jack), then publishes the entire bundle to userspace via `snd_card_register()`. Registration creates the `/sys/class/sound/cardN` device node, populates `/proc/asound/cards`, and unblocks the per-card `/dev/snd/controlCN` character device — the gate that arbitrates every other `/dev/snd/*` minor. On hot-unplug, `snd_card_disconnect()` replaces every open file's `f_op` with a stub (`snd_shutdown_f_ops`) so in-flight ioctls return `-ENODEV` instead of dereferencing freed driver state, walks every registered `snd_device` calling `dev_disconnect`, drains `card->files_list`, then drops the device-model reference. The matching `snd_card_free()` (or refcount-deferred `snd_card_free_when_closed()`) executes `dev_free` on every device, calls `card->private_free`, and releases the slot in the global `snd_cards[]` bitmap.

Per-card invariants critical for v0 FFI: (1) the `snd_cards_lock` bitmap (`SNDRV_CARDS = 32` slots) is the only authority for index allocation; (2) `card->shutdown` is the one-way latch that gates everything between disconnect and free; (3) `card->files_list` plus `card->files_lock` is the audit trail of every userspace handle and the basis for `snd_card_disconnect_sync()` waiting on `card->remove_sleep`. Per-PM, `snd_power_ref_and_wait()` synchronizes control ioctls against `SNDRV_CTL_POWER_D0`. Per-procfs, `snd_info_card_create/register/disconnect/free` chain mirrors the lifecycle exactly.

This Tier-3 covers `sound/core/init.c` (~1176 lines). The `struct snd_device` machinery it depends on lives in `sound/core/device.c` and is documented here as it is part of the same lifecycle contract.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct snd_card` | per-card container | `SndCard` |
| `struct snd_monitor_file` | per-open-fd audit entry | `SndMonitorFile` |
| `struct snd_device` | per-component child object | `SndDevice` |
| `struct snd_device_ops` | per-component vtable (`dev_free` / `dev_register` / `dev_disconnect`) | `SndDeviceOps` |
| `snd_cards[SNDRV_CARDS]` | per-index slot table | `Alsa::CARDS` |
| `snd_cards_lock` | per-index bitmap | `Alsa::CARDS_LOCK` |
| `snd_card_mutex` | per-table-mutate | `Alsa::CARDS_MUTEX` |
| `snd_card_new()` | per-allocate + per-init | `SndCard::new` |
| `snd_devm_card_new()` | per-devres managed variant | `SndCard::devm_new` |
| `snd_card_init()` (static) | per-fill | `SndCard::init` |
| `snd_card_register()` | per-publish | `SndCard::register_` |
| `snd_card_disconnect()` | per-hot-unplug | `SndCard::disconnect` |
| `snd_card_disconnect_sync()` | per-disconnect + wait closes | `SndCard::disconnect_sync` |
| `snd_card_free()` | per-blocking-free | `SndCard::free` |
| `snd_card_free_when_closed()` | per-refcount-deferred-free | `SndCard::free_when_closed` |
| `snd_card_free_on_error()` | per-init-error rollback | `SndCard::free_on_error` |
| `snd_card_ref(idx)` | per-index lookup (refcounted) | `SndCard::by_index` |
| `snd_card_locked(idx)` | per-slot probe | `SndCard::is_locked` |
| `snd_card_set_id()` | per-rename | `SndCard::set_id` |
| `snd_card_add_dev_attr()` | per-sysfs group attach | `SndCard::add_dev_attr` |
| `snd_card_file_add()` | per-fd track | `SndCard::file_add` |
| `snd_card_file_remove()` | per-fd untrack + wake | `SndCard::file_remove` |
| `snd_component_add()` | per-card component-string append | `SndCard::component_add` |
| `snd_device_alloc()` | per-child `struct device` template | `SndCard::device_alloc` |
| `snd_device_new()` | per-component register | `SndDevice::new` |
| `snd_device_free()` | per-component release | `SndDevice::free` |
| `snd_device_register_all()` | per-card iterate `dev_register` | `SndDevice::register_all` |
| `snd_device_disconnect_all()` | per-card iterate `dev_disconnect` | `SndDevice::disconnect_all` |
| `snd_device_free_all()` | per-card iterate `dev_free` | `SndDevice::free_all` |
| `snd_power_ref_and_wait()` | per-PM-D0 wait | `SndPm::ref_and_wait` |
| `snd_power_wait()` | per-PM-D0 wait (legacy form) | `SndPm::wait` |
| `snd_shutdown_f_ops` | per-stub file_operations | `SndCard::SHUTDOWN_FOPS` |

## Compatibility contract

REQ-1: `struct snd_card` essential fields (the FFI cannot relocate these):
- `number: i32` — index into `snd_cards[]` (range 0 .. `SNDRV_CARDS - 1`).
- `id[16]: [u8]` — ASCII identifier, surfaced by `/proc/asound/cards`.
- `driver[16]`, `shortname[32]`, `longname[80]`, `mixername[80]`, `components[128]`.
- `module: *Module` — owning module (locked while card alive).
- `devices: ListHead<SndDevice>` — per-component child list.
- `controls_rwsem: RwSemaphore`, `controls_rwlock: RwLock`, `controls: ListHead`, `ctl_files: ListHead` — per-control mutexes (consumed by `sound/core/control.c`).
- `ctl_numids: XArray`, `ctl_hash: XArray` — per-fast-lookup tables when `CONFIG_SND_CTL_FAST_LOOKUP`.
- `files_lock: SpinLock`, `files_list: ListHead<SndMonitorFile>` — per-open-fd audit.
- `memory_mutex: Mutex` — per-DMA-buffer-allocation serialization.
- `power_sleep: WaitQueueHead`, `power_ref_sleep: WaitQueueHead`, `power_ref: Atomic` — per-PM-D0 sync (`CONFIG_PM`).
- `remove_sleep: WaitQueueHead` — per-disconnect-sync wake on empty `files_list`.
- `card_dev: Device` — per-card embedded struct device (sysfs anchor, `release = release_card_device`).
- `dev_groups[4]: [*AttrGroup; 4]` — per-sysfs attribute slots (`card_dev_attr_group` is slot 0).
- `dev: *Device` — parent device (the bus device passed to `snd_card_new`).
- `private_data: *void`, `private_free: fn(*SndCard)` — per-driver state + destructor.
- `shutdown: bool` — one-way latch set by `snd_card_disconnect`.
- `registered: bool` — true after `device_add(&card_dev)` succeeded.
- `releasing: bool` — set by `snd_card_do_free` for re-entry guard.
- `managed: bool` — set by `snd_devm_card_new` to enable devres path.
- `release_completion: *Completion` — optional sync from `snd_card_free`.
- `sync_irq: i32` — IRQ to `synchronize_irq()` on disconnect.
- `debugfs_root: *Dentry` — `/sys/kernel/debug/<dev>` (when `CONFIG_SND_DEBUG`).
- `irq_descr[32]` — per-IRQ printk prefix.

REQ-2: `struct snd_monitor_file`:
- `file: *File` — opened fd.
- `disconnected_f_op: *FileOperations` — saved real `f_op` before disconnect swap.
- `list: ListHead` — link in `card->files_list`.
- `shutdown_list: ListHead` — link in global `shutdown_files` while disconnected.

REQ-3: `struct snd_device`:
- `list: ListHead` — link in `card->devices`.
- `card: *SndCard` — back-pointer.
- `state: SndDeviceState` — `BUILD` → `REGISTERED` → `DISCONNECTED`.
- `type: SndDeviceType` — `SNDRV_DEV_CONTROL` / `_PCM` / `_RAWMIDI` / `_TIMER` / `_SEQUENCER` / `_HWDEP` / `_JACK` / `_COMPRESS` / `_BUS` / `_CODEC` / `_INFO` / `_LOWLEVEL`.
- `device_data: *void` — per-component object (e.g. `*SndPcm`).
- `ops: *SndDeviceOps` — vtable.

REQ-4: `struct snd_device_ops`:
- `dev_free(dev: *SndDevice) -> i32` — called by `snd_device_free_all` (always).
- `dev_register(dev: *SndDevice) -> i32` — called by `snd_device_register_all` (after `snd_card_register` device_add).
- `dev_disconnect(dev: *SndDevice) -> i32` — called by `snd_device_disconnect_all` (on hot-unplug).

REQ-5: `snd_card_new(parent, idx, xid, module, extra_size, card_ret) -> i32`:
- Validate `card_ret != NULL` (`snd_BUG_ON`).
- `*card_ret = NULL`.
- if `extra_size < 0`: `extra_size = 0`.
- `card = kzalloc(sizeof(snd_card) + extra_size, GFP_KERNEL)`.
- `snd_card_init(card, parent, idx, xid, module, extra_size)`.
- On error: return; `card` already freed by error path or `release_card_device` (managed).
- `*card_ret = card`.
- return 0.

REQ-6: `snd_card_init(card, parent, idx, xid, module, extra_size) -> i32`:
- if `extra_size > 0`: `card->private_data = (char*)card + sizeof(snd_card)`.
- if `xid`: `strscpy(card->id, xid, sizeof card->id)`.
- /* Slot acquisition under `snd_card_mutex` */
- if `idx < 0`: `idx = get_slot_from_bitmask(idx, module_slot_match, module)` (per-module preferred slot).
- if `idx < 0`: `idx = get_slot_from_bitmask(idx, check_empty_slot, module)` (any free).
- if `idx < 0`: `err = -ENODEV`.
- else if `idx < snd_ecards_limit` ∧ `test_bit(idx, snd_cards_lock)`: `err = -EBUSY`.
- else if `idx >= SNDRV_CARDS`: `err = -ENODEV`.
- else: `set_bit(idx, snd_cards_lock)`; `snd_ecards_limit = max(snd_ecards_limit, idx+1)`.
- On err: free `card` if not managed; return.
- `card->dev = parent`; `card->number = idx`; `card->module = module`.
- Init lists/locks/waitqueues per REQ-1.
- `device_initialize(&card->card_dev)`; `parent = parent`; `class = sound_class`; `release = release_card_device`; `groups = card->dev_groups`; `card->dev_groups[0] = &card_dev_attr_group`.
- `kobject_set_name(&card->card_dev.kobj, "card%d", idx)`.
- `snprintf(card->irq_descr, ..., "%s:%s", dev_driver_string(parent), dev_name(&card->card_dev))`.
- `snd_ctl_create(card)` — installs `SNDRV_DEV_CONTROL` device (the gate). On err goto `__error`.
- `snd_info_card_create(card)` — `/proc/asound/cardN`. On err goto `__error_ctl`.
- `#ifdef CONFIG_SND_DEBUG`: `debugfs_create_dir(dev_name(...), sound_debugfs_root)` → `card->debugfs_root`.
- `#ifdef CONFIG_SND_CTL_DEBUG`: kmalloc `card->value_buf`.
- return 0.
- `__error_ctl`: `snd_device_free_all(card)`.
- `__error`: `put_device(&card->card_dev)`; return err.

REQ-7: `snd_devm_card_new(parent, idx, xid, module, extra_size, card_ret) -> i32`:
- `card->managed = true`; attaches `__snd_card_release` via `devm_add_action_or_reset`.
- On parent device removal: `__snd_card_release` → `snd_card_free`.

REQ-8: `snd_card_free_on_error(dev, ret) -> i32`:
- if `ret < 0` ∧ `dev->driver_data is snd_card`: `snd_card_free(card)`.
- Convenience for `probe()` cleanup.

REQ-9: `snd_card_register(card) -> i32`:
- `snd_BUG_ON(!card)`.
- if !`card->registered`:
  - `device_add(&card->card_dev)`. On err return.
  - `card->registered = true`.
- else if `card->managed`:
  - `devm_remove_action(card->dev, trigger_card_free, card)`. (Re-arm below.)
- if `card->managed`:
  - `devm_add_action(card->dev, trigger_card_free, card)`. On err return.
- `snd_device_register_all(card)`. On err return.
- /* Publish under `snd_card_mutex` */:
  - if `snd_cards[card->number] != NULL`: pending re-register — just `snd_info_card_register(card)`.
  - else:
    - if `card->id[0]`: copy `card->id` to scratch; `snd_card_set_id_no_lock(card, scratch, scratch)` (uniqueness).
    - else: derive from `shortname` ∨ `longname` via `retrieve_id_from_card_name`; `snd_card_set_id_no_lock`.
  - `snd_cards[card->number] = card`.
- `snd_info_card_register(card)`. On err return.
- `#if IS_ENABLED(CONFIG_SND_MIXER_OSS)`: `snd_mixer_oss_notify_callback(card, REGISTER)`.
- return 0.

REQ-10: `snd_card_disconnect(card)`:
- if `card == NULL`: return.
- /* Latch under `card->files_lock` */:
  - if `card->shutdown`: return (idempotent).
  - `card->shutdown = 1`.
  - For each `mfile` in `card->files_list`:
    - `mfile->disconnected_f_op = mfile->file->f_op`.
    - Under `shutdown_lock`: `list_add(&mfile->shutdown_list, &shutdown_files)`.
    - `mfile->file->f_op = &snd_shutdown_f_ops`.
    - `fops_get(mfile->file->f_op)` — pin the stub module.
- `#ifdef CONFIG_PM`: `wake_up_all(&card->power_sleep)` — abort any PM waiters first.
- `#if IS_ENABLED(CONFIG_SND_MIXER_OSS)`: `snd_mixer_oss_notify_callback(card, DISCONNECT)`.
- `snd_device_disconnect_all(card)` — iterate `card->devices`, call each `ops->dev_disconnect`.
- if `card->sync_irq > 0`: `synchronize_irq(card->sync_irq)` — drain in-flight IRQ handlers.
- `snd_info_card_disconnect(card)`.
- `#ifdef CONFIG_SND_DEBUG`: `debugfs_remove(card->debugfs_root)`; clear.
- if `card->registered`: `device_del(&card->card_dev)`; `card->registered = false`.
- /* Free slot under `snd_card_mutex` */:
  - `snd_cards[card->number] = NULL`.
  - `clear_bit(card->number, snd_cards_lock)`.
- `snd_power_sync_ref(card)` — wait for in-flight `snd_power_ref` users.

REQ-11: `snd_card_disconnect_sync(card)`:
- `snd_card_disconnect(card)`.
- Under `card->files_lock` (`spin_lock_irq`): `wait_event_lock_irq(card->remove_sleep, list_empty(&card->files_list), card->files_lock)`.

REQ-12: `snd_card_free_when_closed(card)`:
- `snd_card_disconnect(card)`.
- `put_device(&card->card_dev)` — drop init ref; if no userspace fds remain, `release_card_device` → `snd_card_do_free` fires immediately; otherwise deferred until last `snd_card_file_remove`.

REQ-13: `snd_card_free(card)`:
- `release_completion: Completion`.
- if !`card->managed`: `card->release_completion = &release_completion`.
- `snd_card_free_when_closed(card)`.
- if !`card->managed`: `wait_for_completion(&release_completion)`.

REQ-14: `snd_card_do_free(card)` (release callback):
- `card->releasing = true`.
- `#if IS_ENABLED(CONFIG_SND_MIXER_OSS)`: `snd_mixer_oss_notify_callback(card, FREE)`.
- `snd_device_free_all(card)` — iterate `card->devices` calling each `ops->dev_free` (order: non-CONTROL first, then CONTROL last).
- if `card->private_free`: `card->private_free(card)`.
- `#ifdef CONFIG_SND_CTL_DEBUG`: `kfree(card->value_buf)`.
- `snd_info_card_free(card)` (warn on failure but not fatal).
- if `card->release_completion`: `complete(card->release_completion)`.
- if !`card->managed`: `kfree(card)`.

REQ-15: `snd_disconnect_*` stub fops (`snd_shutdown_f_ops`):
- `llseek` → 0.
- `read` → -ENODEV.
- `write` → -ENODEV.
- `poll` → `EPOLLERR | EPOLLNVAL`.
- `unlocked_ioctl` → -ENODEV; ditto `compat_ioctl`.
- `mmap` → -ENODEV.
- `fasync` → -ENODEV.
- `release`: unlink from `shutdown_files`, restore real `f_op` via `fops_put`, call original `release` if any.

REQ-16: `snd_card_file_add(card, file) -> i32`:
- Allocate `mfile`; `mfile->file = file`; init `shutdown_list`.
- Under `card->files_lock`:
  - if `card->shutdown`: free `mfile`; return `-ENODEV`.
  - `list_add(&mfile->list, &card->files_list)`.
  - `get_device(&card->card_dev)` — pin card alive until close.

REQ-17: `snd_card_file_remove(card, file) -> i32`:
- Under `card->files_lock`: scan `files_list` for matching `mfile`; if found, `list_del(&mfile->list)`; under `shutdown_lock` `list_del(&mfile->shutdown_list)`; if `mfile->disconnected_f_op`, `fops_put` it.
- if `list_empty(&card->files_list)`: `wake_up_all(&card->remove_sleep)`.
- if not found: log + return `-ENOENT`.
- `kfree(mfile)`; `put_device(&card->card_dev)`; return 0.

REQ-18: `snd_card_set_id(card, nid)` / `snd_card_set_id_no_lock(card, src, nid)`:
- Sanitize ASCII (letters/digits, leading letter, no '_'... per `card_id_ok`).
- Resolve uniqueness against `snd_cards[]`: append "_N" suffix until unique.
- Update `card->id`; emit `KOBJ_CHANGE` uevent.

REQ-19: `snd_device_new(card, type, device_data, ops) -> i32`:
- Allocate `snd_device`; `state = SNDRV_DEV_BUILD`; init list.
- `list_add` into `card->devices` (ordered so CONTROL is last for `_free_all` ordering).

REQ-20: `snd_device_register_all(card) -> i32`:
- Iterate `card->devices`; for each device in `BUILD`: `ops->dev_register(device)`; on success `state = REGISTERED`; on error: stop and return.

REQ-21: `snd_device_disconnect_all(card)`:
- Iterate `card->devices`; for each in `REGISTERED`: `ops->dev_disconnect(device)`; `state = DISCONNECTED`.

REQ-22: `snd_device_free_all(card)`:
- Iterate `card->devices` twice: first pass skips CONTROL/LOWLEVEL; second pass frees them. Ensures higher-level devices unwind before the control interface that may publish them.

REQ-23: `snd_card_add_dev_attr(card, group) -> i32`:
- Find first NULL slot in `card->dev_groups[0..N-1]`; assign `group`; return 0 or `-ENOSPC`.
- Must be called before `snd_card_register` (groups read by `device_add`).

REQ-24: `snd_component_add(card, component) -> i32`:
- Append `component` to `card->components` (space-separated, max 128 bytes).
- Surfaced in `/proc/asound/cardN/info`.

REQ-25: `snd_card_ref(idx) -> *SndCard`:
- Under `snd_card_mutex`: `card = snd_cards[idx]`; if non-NULL `get_device(&card->card_dev)`; return.
- Caller must `put_device` (i.e. `snd_card_unref`).

REQ-26: `snd_card_locked(card) -> bool`:
- Under `snd_card_mutex`: `test_bit(card, snd_cards_lock)`.

REQ-27: `snd_device_alloc(dev_p, card) -> i32`:
- `*dev_p = NULL`; `dev = kzalloc(sizeof *dev)`; if NULL return `-ENOMEM`.
- `device_initialize(dev)`; if `card`: `dev->parent = &card->card_dev`.
- `dev->class = sound_class`; `dev->release = default_release_alloc`.
- `*dev_p = dev`; return 0.

REQ-28: `snd_power_ref_and_wait(card) -> i32`:
- `snd_power_ref(card)` — `atomic_inc(&card->power_ref)`.
- if `snd_power_get_state(card) == SNDRV_CTL_POWER_D0`: return 0.
- `wait_event_cmd(card->power_sleep, card->shutdown ∨ state == D0, snd_power_unref(card), snd_power_ref(card))`.
- return `card->shutdown ? -ENODEV : 0`.

REQ-29: `/proc/asound/cards` per-line format (read in `snd_card_info_read`):
- `" N [id        ]: driver - shortname\n  longname\n"` per registered card. Format-stable ABI.

REQ-30: `/sys/class/sound/cardN` attributes (from `card_dev_attr_group`):
- `id` (RW), `number` (RO). Extended by `snd_card_add_dev_attr`.

REQ-31: Slot exhaustion semantics:
- `SNDRV_CARDS = 32` slots maximum.
- 33rd registration attempts return `-ENODEV`; printed once via `dev_err`.

REQ-32: Module reference rules:
- `card->module` is the per-card module pointer; `try_module_get` is performed by caller (driver `probe`) not by `snd_card_new` itself.
- `WARN_ON(IS_MODULE(CONFIG_SND) && !module)` enforces non-NULL `module` when ALSA core is modular.

## Acceptance Criteria

- [ ] AC-1: `snd_card_new` returns 0 ∧ `*card_ret != NULL` ∧ `card->number ∈ [0, SNDRV_CARDS)` ∧ `snd_cards_lock[number]` set.
- [ ] AC-2: `snd_card_new(idx=-1, ...)` allocates first matching slot (module preferred), falling through to any empty slot.
- [ ] AC-3: 33rd concurrent live card returns `-ENODEV`.
- [ ] AC-4: `snd_card_register` makes `/dev/snd/controlCN` visible to userspace ∧ `/proc/asound/cardN/` populated ∧ `/sys/class/sound/cardN` present.
- [ ] AC-5: Until `snd_card_register`, `open("/dev/snd/controlCN")` returns `-ENOENT`.
- [ ] AC-6: `snd_card_disconnect` while open fd exists: every subsequent `read`/`write`/`ioctl`/`mmap`/`poll` on that fd returns `-ENODEV` (or `EPOLLERR | EPOLLNVAL`).
- [ ] AC-7: `snd_card_disconnect_sync` blocks until every open fd is closed; returns immediately if none.
- [ ] AC-8: Hot-unplug ordering: `dev_disconnect` called for every `snd_device` before `device_del(&card_dev)`.
- [ ] AC-9: `snd_card_free` is idempotent w.r.t. `snd_card_disconnect` — calling free directly after disconnect does not double-free; calling free without prior disconnect performs disconnect internally.
- [ ] AC-10: `snd_card_free_when_closed` on a card with N open fds defers free until the Nth `snd_card_file_remove`; reference count on `card_dev` reflects this.
- [ ] AC-11: `snd_devm_card_new`: parent device `remove()` triggers `snd_card_free` automatically.
- [ ] AC-12: `snd_card_set_id` rejects non-ASCII / non-letter-leading IDs and de-duplicates against `snd_cards[]`.
- [ ] AC-13: `snd_card_add_dev_attr` returns `-ENOSPC` after `len(dev_groups) - 1` (= 3) groups assigned.
- [ ] AC-14: Power state: `snd_power_ref_and_wait` returns `-ENODEV` if `card->shutdown` is latched while waiting.
- [ ] AC-15: `snd_device_free_all` invokes CONTROL device `dev_free` strictly last; all other types free before CONTROL.

## Architecture

```
struct SndCard {
  number: i32,
  id: [u8; 16],
  driver: [u8; 16],
  shortname: [u8; 32],
  longname: [u8; 80],
  mixername: [u8; 80],
  components: [u8; 128],
  module: *Module,
  devices: ListHead<SndDevice>,           // anchored by snd_device_new
  controls_rwsem: RwSemaphore,
  controls_rwlock: RwLock,
  controls: ListHead,
  ctl_files: ListHead,
  ctl_numids: XArray,                     // CONFIG_SND_CTL_FAST_LOOKUP
  ctl_hash: XArray,
  files_lock: SpinLock,
  files_list: ListHead<SndMonitorFile>,
  memory_mutex: Mutex,
  power_sleep: WaitQueueHead,             // CONFIG_PM
  power_ref_sleep: WaitQueueHead,
  power_ref: AtomicI32,
  remove_sleep: WaitQueueHead,
  card_dev: Device,                       // embedded; sysfs anchor
  dev_groups: [*AttrGroup; 4],            // slot 0 = card_dev_attr_group
  dev: *Device,                           // parent (bus device)
  private_data: *void,
  private_free: Option<fn(*SndCard)>,
  shutdown: bool,                         // one-way latch
  registered: bool,
  releasing: bool,
  managed: bool,
  release_completion: *Completion,
  sync_irq: i32,
  debugfs_root: *Dentry,
  irq_descr: [u8; 32],
}

struct SndMonitorFile {
  file: *File,
  disconnected_f_op: *FileOperations,
  list: ListHead,                         // in card->files_list
  shutdown_list: ListHead,                // in global shutdown_files
}

struct SndDevice {
  list: ListHead,                         // in card->devices
  card: *SndCard,
  state: SndDeviceState,                  // BUILD | REGISTERED | DISCONNECTED
  type_: SndDeviceType,                   // CONTROL | PCM | RAWMIDI | TIMER | SEQUENCER | HWDEP | JACK | COMPRESS | BUS | CODEC | INFO | LOWLEVEL
  device_data: *void,
  ops: *SndDeviceOps,
}

struct SndDeviceOps {
  dev_free: Option<fn(*SndDevice) -> i32>,
  dev_register: Option<fn(*SndDevice) -> i32>,
  dev_disconnect: Option<fn(*SndDevice) -> i32>,
}

const SNDRV_CARDS: usize = 32;
static CARDS: [Option<*SndCard>; SNDRV_CARDS];
static CARDS_LOCK: BitMap<SNDRV_CARDS>;
static CARDS_MUTEX: Mutex;
static SHUTDOWN_FILES: ListHead;
static SHUTDOWN_LOCK: SpinLock;
```

`SndCard::new(parent, idx, xid, module, extra_size) -> Result<*SndCard, Errno>`:
1. /* Validate */
2. Allocate `card` = `kzalloc(sizeof(SndCard) + extra_size, GFP_KERNEL)`.
3. `SndCard::init(card, parent, idx, xid, module, extra_size)`. On error: `init` already freed `card` (or scheduled via release) — propagate.
4. Return `card`.

`SndCard::init(card, parent, idx, xid, module, extra_size) -> Result<(), Errno>`:
1. /* Wire `private_data` */
2. if `extra_size > 0`: `card.private_data = (card as *u8).add(sizeof SndCard) as *void`.
3. /* Copy id */
4. if `xid`: `strscpy(card.id, xid, 16)`.
5. /* Slot acquisition */
6. lock CARDS_MUTEX:
   - if `idx < 0`: try module preferred slot.
   - if `idx < 0`: try first empty slot.
   - if `idx < 0`: err = -ENODEV.
   - else if `idx < snd_ecards_limit && CARDS_LOCK.test(idx)`: err = -EBUSY.
   - else if `idx >= SNDRV_CARDS`: err = -ENODEV.
   - else: `CARDS_LOCK.set(idx)`; `snd_ecards_limit = max(_, idx+1)`.
7. if err: if !card.managed kfree(card); return err.
8. card.dev = parent; card.number = idx; card.module = module.
9. Init lists, locks, waitqueues, atomic, sync_irq = -1.
10. `device_initialize(&card.card_dev)`; set parent/class/release/groups.
11. `kobject_set_name(&card.card_dev.kobj, "card{idx}")`.
12. Build `card.irq_descr`.
13. `snd_ctl_create(card)` → installs CONTROL device. On err goto __error.
14. `snd_info_card_create(card)`. On err goto __error_ctl.
15. (debugfs / value_buf compile-time-conditional.)
16. Return Ok.
17. __error_ctl: `SndDevice::free_all(card)`.
18. __error: `put_device(&card.card_dev)`; return err.

`SndCard::register_(card) -> Result<(), Errno>`:
1. /* Publish device-model node */
2. if !card.registered:
   - `device_add(&card.card_dev)`. On err return.
   - card.registered = true.
3. else if card.managed: rearm devres action (remove then re-add below).
4. if card.managed: `devm_add_action(card.dev, trigger_card_free, card)`. On err return.
5. /* Register children */
6. `SndDevice::register_all(card)`. On err return.
7. /* Publish in global table under CARDS_MUTEX */
8. lock CARDS_MUTEX:
   - if CARDS[card.number].is_some(): /* re-register */ goto info_register.
   - if card.id[0] != 0: `SndCard::set_id_no_lock(card, card.id, card.id)`.
   - else: derive from shortname/longname, then set_id_no_lock.
   - CARDS[card.number] = Some(card).
9. info_register: `snd_info_card_register(card)`. On err return.
10. (Optional OSS mixer notify.)
11. Return Ok.

`SndCard::disconnect(card)`:
1. if card.is_null(): return.
2. /* Replace f_op of every open fd */
3. lock card.files_lock:
   - if card.shutdown: return (idempotent).
   - card.shutdown = true.
   - for mfile in card.files_list:
     - mfile.disconnected_f_op = mfile.file.f_op.
     - under SHUTDOWN_LOCK: SHUTDOWN_FILES.add(&mfile.shutdown_list).
     - mfile.file.f_op = &SHUTDOWN_FOPS.
     - `fops_get(mfile.file.f_op)`.
4. /* Wake PM waiters first to avoid dead-lock vs kctl */
5. wake_up_all(&card.power_sleep).
6. /* OSS mixer notify (DISCONNECT). */
7. `SndDevice::disconnect_all(card)`.
8. if card.sync_irq > 0: `synchronize_irq(card.sync_irq)`.
9. `snd_info_card_disconnect(card)`.
10. debugfs_remove(card.debugfs_root); card.debugfs_root = null.
11. if card.registered: `device_del(&card.card_dev)`; card.registered = false.
12. /* Release slot */
13. lock CARDS_MUTEX: CARDS[card.number] = None; CARDS_LOCK.clear(card.number).
14. `snd_power_sync_ref(card)`.

`SndCard::disconnect_sync(card)`:
1. `SndCard::disconnect(card)`.
2. `wait_event_lock_irq(card.remove_sleep, card.files_list.is_empty(), card.files_lock)`.

`SndCard::free_when_closed(card)`:
1. `SndCard::disconnect(card)`.
2. `put_device(&card.card_dev)` — drop the init ref; release fires now if no outstanding `snd_card_file_add` references, or later on the last `snd_card_file_remove`.

`SndCard::free(card)`:
1. `release_completion: Completion`.
2. if !card.managed: card.release_completion = &release_completion.
3. `SndCard::free_when_closed(card)`.
4. if !card.managed: `wait_for_completion(&release_completion)`.

`SndCard::do_free(card)` (release callback, never called directly):
1. card.releasing = true.
2. OSS mixer notify FREE.
3. `SndDevice::free_all(card)`.
4. card.private_free.map(|f| f(card)).
5. `snd_info_card_free(card)` (warn on err, non-fatal).
6. if card.release_completion: `complete(card.release_completion)`.
7. if !card.managed: `kfree(card)`.

`SndCard::file_add(card, file) -> Result<(), Errno>`:
1. Allocate `mfile`.
2. Lock card.files_lock:
   - if card.shutdown: free mfile; return -ENODEV.
   - card.files_list.add(&mfile.list).
   - `get_device(&card.card_dev)`.
3. Return Ok.

`SndCard::file_remove(card, file) -> Result<(), Errno>`:
1. Lock card.files_lock:
   - Walk files_list; on match: del from list; under SHUTDOWN_LOCK del from shutdown_list; fops_put disconnected_f_op; found = mfile; break.
   - if card.files_list.is_empty(): wake_up_all(&card.remove_sleep).
2. if !found: log; return -ENOENT.
3. kfree(found); `put_device(&card.card_dev)`; return Ok.

`SndDevice::new(card, type_, device_data, ops) -> Result<(), Errno>`:
1. Allocate `dev`; populate fields; state = BUILD.
2. Splice into card.devices in canonical order (CONTROL goes last).
3. Return Ok.

`SndDevice::register_all(card) -> Result<(), Errno>`:
1. for dev in card.devices where dev.state == BUILD:
   - if dev.ops.dev_register.is_some(): err = dev.ops.dev_register(dev); if err: return err; dev.state = REGISTERED.
2. Return Ok.

`SndDevice::disconnect_all(card)`:
1. for dev in card.devices where dev.state == REGISTERED:
   - if dev.ops.dev_disconnect.is_some(): dev.ops.dev_disconnect(dev).
   - dev.state = DISCONNECTED.

`SndDevice::free_all(card)`:
1. Pass 1: for dev in card.devices where dev.type_ ∉ {CONTROL, LOWLEVEL}: invoke dev.ops.dev_free; remove + free dev.
2. Pass 2: same for {CONTROL, LOWLEVEL}.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `slot_bitmap_invariant` | INVARIANT | per-`SndCard::init` success: `CARDS_LOCK.test(card.number)` ∧ card.number < SNDRV_CARDS. |
| `slot_freed_on_disconnect` | INVARIANT | per-`SndCard::disconnect`: post: !CARDS_LOCK.test(card.number) ∧ CARDS[card.number].is_none(). |
| `shutdown_one_way_latch` | INVARIANT | once `card.shutdown == true`, no transition back to false. |
| `files_list_drained_for_sync` | INVARIANT | post-`disconnect_sync`: `card.files_list.is_empty()`. |
| `fop_swap_under_files_lock` | INVARIANT | per-`disconnect`: every `mfile.file.f_op = &SHUTDOWN_FOPS` swap occurs while files_lock held. |
| `cards_table_mutated_under_mutex` | INVARIANT | every read/write of `CARDS[idx]` and `CARDS_LOCK` outside init happens under CARDS_MUTEX. |
| `dev_groups_bounded` | INVARIANT | `add_dev_attr` cannot overflow `dev_groups`; returns -ENOSPC at slot N-1. |
| `register_idempotent_if_managed` | INVARIANT | calling `register_` twice on a managed card: devres action re-armed exactly once. |

### Layer 2: TLA+

`sound/core/card-lifecycle.tla` states: ALLOC → INIT → BUILD → REGISTERED → DISCONNECTED → FREED, plus parallel files_list mutation. Actions:
- `New` (slot acquired, ctl created, info created).
- `Register` (device_add, register_all, set_id, publish in CARDS).
- `FileAdd` / `FileRemove` (concurrent with everything except FREED).
- `Disconnect` (swap f_op, disconnect_all, device_del, clear slot).
- `DisconnectSync` (Disconnect ∧ wait files empty).
- `FreeWhenClosed` (Disconnect ∧ put_device).
- `Free` (FreeWhenClosed ∧ wait completion).
- `DoFree` (free_all, private_free, kfree).

Properties:
- `safety_no_register_without_init` — every Register transition predecessor is BUILD.
- `safety_no_dev_register_after_disconnect` — Disconnect strictly precedes any FREED transition for any child device.
- `safety_files_swapped_before_dev_disconnect` — within Disconnect, fop swap completes before disconnect_all to prevent fd-side races with mid-disconnect ioctls.
- `safety_slot_freed_exactly_once` — CARDS_LOCK transitions set→clear are paired 1:1 with New→Disconnect.
- `liveness_disconnect_sync_terminates` — given no leaked fds, DisconnectSync eventually completes.
- `liveness_free_when_closed_completes` — Free terminates iff all fds eventually close.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SndCard::new` post: card.number ∈ [0, SNDRV_CARDS) ∧ CARDS_LOCK.test(number) | `SndCard::new` |
| `SndCard::register_` post: card.registered ∧ snd_cards[number] == Some(card) ∧ all children REGISTERED | `SndCard::register_` |
| `SndCard::disconnect` post: card.shutdown ∧ !card.registered ∧ all children DISCONNECTED ∧ slot freed | `SndCard::disconnect` |
| `SndCard::file_add` post: list contains mfile ∧ device refcount +1; or returns -ENODEV iff shutdown | `SndCard::file_add` |
| `SndCard::file_remove` post: list does not contain file ∧ device refcount -1 ∧ if list empty, remove_sleep waker fires | `SndCard::file_remove` |
| `SndDevice::free_all` post: CONTROL freed strictly after every other type | `SndDevice::free_all` |
| `SndCard::set_id_no_lock` post: card.id unique within CARDS, ASCII-letters-leading | `SndCard::set_id_no_lock` |

### Layer 4: Verus/Creusot functional

`SndCard::new → register_ → disconnect → free` equivalence to the upstream C lifecycle, witnessed by the alsa-lib regression suite (`alsa-utils`, `alsa-info`, `aplay`, `arecord` open/close cycles), plus `udevadm trigger` enumeration of `/sys/class/sound/cardN`, `cat /proc/asound/cards`, `cat /proc/asound/version`. Hot-unplug parity validated by USB-audio surprise-removal regression (matches upstream sequence of `EPOLLERR` delivery, `-ENODEV` on subsequent ioctls, eventual fd close, eventual card free).

## Hardening

(Inherits row-1 features from `sound/00-overview.md` § Hardening.)

ALSA card-init reinforcement:

- **Per-`card->shutdown` one-way latch** — defense against per-double-disconnect and per-late-arrival ioctl racing free.
- **Per-`f_op` swap under `files_lock`** — defense against per-disconnect-vs-read TOCTOU on `/dev/snd/*`.
- **Per-`snd_shutdown_f_ops` returns `-ENODEV`** — defense against per-stale-fd kernel pointer follow.
- **Per-`fops_get` on shutdown stub** — defense against per-module-unload while disconnected fd still open.
- **Per-`snd_card_mutex` guards `snd_cards[]` and `snd_cards_lock`** — defense against per-double-slot-allocation race.
- **Per-`card_dev` device-model refcount + `release_card_device`** — defense against per-premature free with outstanding userspace handles.
- **Per-`SNDRV_CARDS = 32` bound** — defense against per-card-table-overflow.
- **Per-PM `wake_up_all(power_sleep)` before `disconnect_all`** — defense against per-deadlock with kctl pending on D0.
- **Per-`sync_irq` flush on disconnect** — defense against per-IRQ-handler executing after device gone.
- **Per-`device_del` strictly after `disconnect_all`** — defense against per-sysfs-unlink-before-children.
- **Per-`SndDevice::free_all` two-pass with CONTROL last** — defense against per-control-freed-while-PCM-still-alive UAF.
- **Per-`snd_BUG_ON(!card_ret)` defensive ABI** — defense against per-driver passing NULL out-param.
- **Per-`WARN_ON(!module)` when CONFIG_SND modular** — defense against per-card outliving its driver module.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — card-info `snd_ctl_card_info` / `snd_pcm_info` returned via copy_to_user bounded by fixed struct sizes; card-id strings `snd_strlcpy_pad` zero-padded before publication.
- **PAX_KERNEXEC** — `snd_card_register`, `snd_card_free`, `snd_card_disconnect`, and the per-device-type register vector reside in RX `.text`; per-device free callbacks (`snd_device_ops.dev_free`) dispatched through kCFI.
- **PAX_RANDKSTACK** — kstack-offset randomization on every `/dev/snd/*` open path entering card lookup.
- **PAX_REFCOUNT** — saturating refcount on `struct snd_card` (`refcount`, `files_count`); defense against driver-vs-userspace `snd_card_free_when_closed` lifetime races wrapping the count.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `snd_card` and per-device security-relevant buffers (PCM, control, rawmidi, seq, timer instances) so disconnected-card buffers cannot be recovered.
- **PAX_UDEREF (SMAP/SMEP)** — ASM_CLAC on every card-ioctl path; sysfs / procfs attribute writes bounded.
- **PAX_RAP / kCFI** — `snd_device_ops` (`dev_free`, `dev_register`, `dev_disconnect`) and `snd_card_ops` dispatched via kCFI-signed indirect calls.
- **GRKERNSEC_HIDESYM** — `snd_cards[]`, `shutdown_files`, and `snd_card_lock` symbols stripped from /proc/kallsyms for non-CAP_SYSLOG.
- **GRKERNSEC_DMESG** — card-register / disconnect / free diagnostics restricted to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict on /dev/snd/* mknod** — devnode creation through `snd_device_alloc` requires `mode = 0660 root:audio` by default; userspace must hold CAP_SYS_ADMIN to override permissions via udev rules.
- **Two-pass `SndDevice::free_all` with CONTROL last** — defense against control-freed-while-PCM-still-alive UAF.
- **`WARN_ON(!module)` on modular CONFIG_SND** — defense against card outliving its driver module (refcount on `snd_card->module`).

Per-doc rationale: sound/core/init.c owns `snd_card` lifecycle — the parent object for every PCM, mixer, MIDI, sequencer, and timer instance. A REFCOUNT wrap or MEMORY_SANITIZE miss here cascades into UAF in every other ALSA submodule. The two-pass free ordering + saturating refcount + per-CPU TSAN bound on device-list mutation make `snd_card` lifetime auditable.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `sound/core/control.c` — `/dev/snd/controlCN` ioctl surface (covered separately in `control.md` Tier-3)
- `sound/core/pcm.c` — PCM device lifecycle (covered separately in `pcm.md` Tier-3)
- `sound/core/info.c` — `/proc/asound/*` text infrastructure (covered separately if expanded)
- `sound/core/device.c` mechanics beyond what `init.c` invokes (the `snd_device_*` ops dispatch loops; documented here only as the contract surface)
- `sound/core/sound.c` — `sound_class`, minor allocation, `snd_register_device` (covered separately if expanded)
- Per-bus driver probe/remove glue (PCI, USB, platform) — out of scope at the core layer
- Implementation code
