# Tier-3: sound/core/pcm.c — ALSA PCM (Pulse Code Modulation) core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: sound/00-overview.md
upstream-paths:
  - sound/core/pcm.c (~1232 lines)
  - sound/core/pcm_native.c
  - sound/core/pcm_memory.c
  - include/sound/pcm.h
  - include/uapi/sound/asound.h
-->

## Summary

`sound/core/pcm.c` is the PCM core's lifecycle / topology / procfs layer. It owns construction of `struct snd_pcm`, its two `struct snd_pcm_str` direction containers (PLAYBACK / CAPTURE), and the linked list of `struct snd_pcm_substream` (subdevices) under each direction. The PCM itself is exposed to ALSA via a `struct snd_device` of type `SNDRV_DEV_PCM` with three ops: `dev_free`, `dev_register`, `dev_disconnect`. Registration creates `/dev/snd/pcmCNDDp` and `/dev/snd/pcmCNDDc` minors (via `snd_register_device` with `snd_pcm_f_ops[]`), `/proc/asound/cardN/pcmNDx/{info,sub0/info,sub0/hw_params,sub0/sw_params,sub0/status,sub0/prealloc,...}` entries, and per-substream timers (`snd_pcm_timer_init`).

`snd_pcm_attach_substream` is the per-open hot path: it walks `pstr->substream` for a free slot honoring `SNDRV_PCM_INFO_HALF_DUPLEX`, `O_APPEND`, and `prefer_subdevice`, allocates `struct snd_pcm_runtime`, allocates `runtime->status` and `runtime->control` as page-exact pages (these are the mmap-able status/control regions), initializes `runtime->sleep` and `runtime->tsleep` waitqueues, sets state to `SNDRV_PCM_STATE_OPEN`, and hands the substream to the caller (`snd_pcm_open` from `pcm_native.c`). `snd_pcm_detach_substream` reverses it on `close()`.

The PCM state machine (`OPEN → SETUP → PREPARED → RUNNING ↔ PAUSED → DRAINING → SETUP`, plus `XRUN`, `SUSPENDED`, `DISCONNECTED`) is driven by `snd_pcm_*` operations in `pcm_native.c`, but the state field itself lives in `runtime->status->state` (mmap-visible) and `runtime->state` (kernel copy); `pcm.c` exposes `__snd_pcm_set_state` to mutate them coherently and is the authority for setting `OPEN` (open) and `DISCONNECTED` (hot-unplug).

This Tier-3 covers `sound/core/pcm.c` (~1232 lines). The PCM-state transition operators (`snd_pcm_hw_params` / `_sw_params` / `_prepare` / `_trigger` / `_drain` / `_mmap` / `_drop` etc.) live in `pcm_native.c` and are documented at the contract level here because they consume the structs `pcm.c` owns.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct snd_pcm` | per-PCM-instance container | `SndPcm` |
| `struct snd_pcm_str` | per-direction (P/C) container | `SndPcmStr` |
| `struct snd_pcm_substream` | per-subdevice instance | `SndPcmSubstream` |
| `struct snd_pcm_runtime` | per-open runtime state | `SndPcmRuntime` |
| `struct snd_pcm_ops` | per-substream driver vtable | `SndPcmOps` |
| `struct snd_pcm_notify` | OSS-emulation notify hook | `SndPcmNotify` |
| `snd_pcm_devices` | per-global PCM list | `Pcm::DEVICES` |
| `snd_pcm_notify_list` | per-global notify list | `Pcm::NOTIFY_LIST` |
| `register_mutex` | per-global add/remove serializer | `Pcm::REGISTER_MUTEX` |
| `snd_pcm_new()` | per-create | `SndPcm::new` |
| `snd_pcm_new_internal()` | per-create-internal (no userspace) | `SndPcm::new_internal` |
| `_snd_pcm_new()` (static) | per-shared constructor | `SndPcm::new_inner` |
| `snd_pcm_new_stream()` | per-direction populate | `SndPcm::new_stream` |
| `snd_pcm_attach_substream()` | per-open substream pick + runtime alloc | `SndPcmSubstream::attach` |
| `snd_pcm_detach_substream()` | per-close release | `SndPcmSubstream::detach` |
| `snd_pcm_free()` (static) | per-free | `SndPcm::free` |
| `snd_pcm_free_stream()` (static) | per-direction free | `SndPcm::free_stream` |
| `snd_pcm_dev_free()` (static) | `snd_device_ops.dev_free` adapter | `SndPcm::dev_free` |
| `snd_pcm_dev_register()` (static) | `snd_device_ops.dev_register` adapter | `SndPcm::dev_register` |
| `snd_pcm_dev_disconnect()` (static) | `snd_device_ops.dev_disconnect` adapter | `SndPcm::dev_disconnect` |
| `snd_pcm_add()` (static) | per-global-list insert (sorted) | `SndPcm::add_to_list` |
| `snd_pcm_get()` (static) | per-card lookup by device | `SndPcm::lookup` |
| `snd_pcm_next()` (static) | per-iter next free device | `SndPcm::next_device` |
| `snd_pcm_notify()` | per-notify subscribe/unsubscribe (OSS) | `SndPcm::notify` |
| `snd_pcm_format_name()` | per-format string | `SndPcm::format_name` |
| `snd_pcm_proc_read()` | per-`/proc/asound/pcm` callback | `SndPcm::proc_read` |
| `snd_pcm_proc_init()` (static) | per-`/proc/asound/pcm` register | `SndPcm::proc_init` |
| `snd_pcm_proc_done()` (static) | per-`/proc/asound/pcm` unregister | `SndPcm::proc_done` |
| `snd_pcm_control_ioctl()` (static) | per-controlCN PCM ioctl | `SndPcm::control_ioctl` |
| `snd_pcm_lib_preallocate_pages_for_all()` | per-PCM DMA prealloc (legacy) | `SndPcmMem::preallocate_for_all` |
| `snd_pcm_set_managed_buffer_all()` | per-PCM managed buffer (preferred) | `SndPcmMem::set_managed_for_all` |
| `snd_pcm_set_ops()` | per-direction install `snd_pcm_ops` | `SndPcm::set_ops` |
| `__snd_pcm_set_state()` | per-state-write (status mirror) | `SndPcmRuntime::set_state` |
| `snd_pcm_running()` | per-state predicate | `SndPcmRuntime::running` |
| `snd_pcm_stop()` | per-stop (XRUN/disconnect) | `SndPcmSubstream::stop` |
| `snd_pcm_sync_stop()` | per-drain stop callbacks | `SndPcmSubstream::sync_stop` |
| `snd_pcm_group_init()` | per-substream-group init | `SndPcmGroup::init` |

## Compatibility contract

REQ-1: `struct snd_pcm` essential fields:
- `card: *SndCard` — owning card.
- `device: i32` — per-card device index (0..N).
- `id[64]: [u8]` — text identifier.
- `streams[2]: [SndPcmStr; 2]` — index 0 = PLAYBACK, 1 = CAPTURE.
- `info_flags: u32` — `SNDRV_PCM_INFO_*` (HALF_DUPLEX, NONINTERLEAVED, MMAP, MMAP_VALID, ...).
- `dev_class: u32` — `SNDRV_PCM_CLASS_{GENERIC,MULTI,MODEM,DIGITIZER}`.
- `dev_subclass: u32` — `SNDRV_PCM_SUBCLASS_*`.
- `internal: bool` — no userspace exposure (ASoC BE-PCM).
- `nonatomic: bool` — uses mutex instead of spinlock for stream lock.
- `no_device_suspend: bool` — skip device-level suspend.
- `open_mutex: Mutex` — per-PCM open/close serialization.
- `open_wait: WaitQueueHead` — `O_NONBLOCK==0` open waiters.
- `list: ListHead` — link in global `snd_pcm_devices`.
- `private_data: *void`, `private_free: fn(*SndPcm)`.

REQ-2: `struct snd_pcm_str`:
- `stream: i32` — `SNDRV_PCM_STREAM_PLAYBACK` (0) or `_CAPTURE` (1).
- `substream_count: u32` — declared substreams.
- `substream_opened: u32` — running counter (`attach_substream` ++ / `detach_substream` --).
- `substream: *SndPcmSubstream` — head of singly-linked substream list (via `.next`).
- `chmap_kctl: *SndKctl` — per-channel-map control (if any).
- `dev: *Device` — per-direction `/sys/class/sound/pcmCNDDx`.
- `pcm: *SndPcm` — back-pointer.
- `oss: SndPcmOssStream` — OSS emulation state (`CONFIG_SND_PCM_OSS`).
- per-procfs handles: `proc_root`, `proc_info_entry`, `proc_xrun_injection_entry`, `proc_xrun_debug_entry`.

REQ-3: `struct snd_pcm_substream`:
- `pcm: *SndPcm`, `pstr: *SndPcmStr`.
- `number: u32` — subdevice index within direction.
- `stream: i32` — direction copy.
- `name[32]: [u8]` — `"subdevice #N"`.
- `next: *SndPcmSubstream` — next in pstr list.
- `ops: *SndPcmOps` — driver vtable.
- `runtime: *SndPcmRuntime` — non-NULL between open/close.
- `private_data: *void` — copied from `pcm->private_data` at attach.
- `ref_count: i32` — guarded by `pcm->open_mutex`; >=1 while open; `O_APPEND` re-uses (++ instead of fresh runtime).
- `f_flags: u32` — file `O_*` flags at open.
- `pid: *Pid` — `task_pid(current)` at open.
- `group: *SndPcmGroup`, `self_group: SndPcmGroup`, `link_list: ListHead` — substream linking (group sync).
- `mmap_count: AtomicI32` — `vm_open/close` counter.
- `buffer_bytes_max: u32` — driver-declared cap (default `UINT_MAX`).
- `pcm_release: fn(*SndPcmSubstream)` — extra release hook (rarely set).
- `timer: *SndTimer` — per-substream timer (`snd_pcm_timer_init`).
- `xrun_counter: u32` — `CONFIG_SND_PCM_XRUN_DEBUG`.

REQ-4: `struct snd_pcm_runtime` (key fields):
- `state: snd_pcm_state_t` — driver-side copy.
- `suspended_state: snd_pcm_state_t` — saved across suspend.
- `status: *SndPcmMmapStatus` — page-allocated; mmap-able RO by userspace; contains `state`, `hw_ptr`, `tstamp`, `audio_tstamp`.
- `control: *SndPcmMmapControl` — page-allocated; mmap-able RW; contains `appl_ptr`, `avail_min`.
- `sleep: WaitQueueHead`, `tsleep: WaitQueueHead` — read/write blocking; transfer-sync.
- `buffer_mutex: Mutex` — guards `dma_area` swap during hw_params/hw_free.
- `buffer_accessing: Atomic` — pin against concurrent hw_params.
- `hw: SndPcmHardware` — driver capabilities (rates, formats, channels, buffer/period bounds).
- `hw_constraints: SndPcmHwConstraints` — `rules` list + masks for `hw_params` refinement.
- `format`, `access`, `subformat`, `rate`, `channels`, `period_size`, `period_count`, `buffer_size` — set by hw_params.
- `frame_bits`, `sample_bits`, `byte_align` — derived.
- `dma_buffer_p: *SndDmaBuffer`, `dma_area: *void`, `dma_addr: dma_addr_t`, `dma_bytes: usize` — buffer.
- `private_data: *void`, `private_free: fn(*SndPcmRuntime)` — driver runtime extension.
- `fasync: *SndFasync` — SIGIO.
- `xrun_threshold`, `stop_threshold`, `silence_threshold`, `silence_size` — sw_params.
- `boundary: u32` — wrap modulus for appl_ptr/hw_ptr.
- `oss: SndPcmOssRuntime` — OSS emulation.

REQ-5: `struct snd_pcm_ops` (driver vtable, consumed by `pcm_native.c`):
- `open(substream) -> i32` — driver setup; assign `runtime->hw`, install constraints.
- `close(substream) -> i32` — driver teardown.
- `ioctl(substream, cmd, arg) -> i32` — usually `snd_pcm_lib_ioctl`.
- `hw_params(substream, params) -> i32` — allocate / reconfigure DMA buffer.
- `hw_free(substream) -> i32` — release DMA buffer.
- `prepare(substream) -> i32` — reset hw pointer; ready for trigger.
- `trigger(substream, cmd) -> i32` — START/STOP/PAUSE_PUSH/PAUSE_RELEASE/SUSPEND/RESUME/DRAIN.
- `sync_stop(substream) -> i32` — wait for in-flight DMA after stop.
- `pointer(substream) -> snd_pcm_uframes_t` — current hw ptr.
- `get_time_info`, `fill_silence`, `copy`, `page`, `mmap`, `ack` — optional.

REQ-6: PCM state machine (`snd_pcm_state_t`, UAPI ABI-fixed):
- `SNDRV_PCM_STATE_OPEN = 0` — set by `snd_pcm_attach_substream`.
- `SNDRV_PCM_STATE_SETUP = 1` — after successful `hw_params`.
- `SNDRV_PCM_STATE_PREPARED = 2` — after `prepare`.
- `SNDRV_PCM_STATE_RUNNING = 3` — after `trigger(START)`.
- `SNDRV_PCM_STATE_XRUN = 4` — underrun (playback) / overrun (capture).
- `SNDRV_PCM_STATE_DRAINING = 5` — playback drain in progress.
- `SNDRV_PCM_STATE_PAUSED = 6` — after `trigger(PAUSE_PUSH)`.
- `SNDRV_PCM_STATE_SUSPENDED = 7` — system suspend.
- `SNDRV_PCM_STATE_DISCONNECTED = 8` — hot-unplug; terminal except for close.
- `SNDRV_PCM_STATE_LAST = DISCONNECTED`.

Valid transitions (from `pcm_native.c`, enforced by callers; `pcm.c` only enters `OPEN` and `DISCONNECTED`):
- OPEN → SETUP (`hw_params`).
- SETUP → SETUP (`hw_params` again with PREPARED reset to SETUP first).
- SETUP → PREPARED (`prepare`).
- PREPARED → RUNNING (`trigger START`).
- RUNNING → XRUN (driver / hw_ptr-vs-appl_ptr violation).
- RUNNING → PAUSED (`trigger PAUSE_PUSH`).
- PAUSED → RUNNING (`trigger PAUSE_RELEASE`).
- RUNNING → DRAINING (`drain`, playback only).
- DRAINING → SETUP (drain complete).
- RUNNING / XRUN / PREPARED / DRAINING → PREPARED (`prepare`).
- any → SUSPENDED (`suspend`).
- SUSPENDED → suspended_state (`resume`).
- any → DISCONNECTED (`dev_disconnect`); only `close` valid afterwards.

REQ-7: `_snd_pcm_new(card, id, device, playback_count, capture_count, internal, rpcm) -> i32`:
- Define ops vtables:
  - public: `{ dev_free = snd_pcm_dev_free, dev_register = snd_pcm_dev_register, dev_disconnect = snd_pcm_dev_disconnect }`.
  - internal: `{ dev_free = snd_pcm_dev_free }` only.
- Validate `card != NULL`.
- if `rpcm`: `*rpcm = NULL`.
- `pcm = kzalloc(sizeof(snd_pcm))`.
- Fill `pcm->card`, `device`, `internal`.
- `mutex_init(&pcm->open_mutex)`; `init_waitqueue_head(&pcm->open_wait)`; `INIT_LIST_HEAD(&pcm->list)`.
- if `id`: `strscpy(pcm->id, id, sizeof pcm->id)`.
- `snd_pcm_new_stream(pcm, SNDRV_PCM_STREAM_PLAYBACK, playback_count)`. On err goto free_pcm.
- `snd_pcm_new_stream(pcm, SNDRV_PCM_STREAM_CAPTURE, capture_count)`. On err goto free_pcm.
- `snd_device_new(card, SNDRV_DEV_PCM, pcm, internal ? &internal_ops : &ops)`. On err goto free_pcm.
- if `rpcm`: `*rpcm = pcm`.
- return 0.
- `free_pcm: snd_pcm_free(pcm); return err`.

REQ-8: `snd_pcm_new(card, id, device, playback_count, capture_count, rpcm)`: thin wrapper, `internal=false`.

REQ-9: `snd_pcm_new_internal(...)`: thin wrapper, `internal=true`; no procfs, no `/dev/snd` minor.

REQ-10: `snd_pcm_new_stream(pcm, stream, substream_count) -> i32`:
- `pstr = &pcm->streams[stream]`.
- `pstr->stream = stream`; `pstr->pcm = pcm`; `pstr->substream_count = substream_count`.
- if `substream_count == 0`: return 0.
- `snd_device_alloc(&pstr->dev, pcm->card)`. On err return.
- `dev_set_name(pstr->dev, "pcmC%iD%i%c", pcm->card->number, pcm->device, stream == PLAYBACK ? 'p' : 'c')`.
- `pstr->dev->groups = pcm_dev_attr_groups`; `pstr->dev->type = &pcm_dev_type`.
- `dev_set_drvdata(pstr->dev, pstr)`.
- if !`pcm->internal`: `snd_pcm_stream_proc_init(pstr)`. On err return.
- For idx in 0..substream_count:
  - `substream = kzalloc(sizeof(SndPcmSubstream))`. On NULL return -ENOMEM.
  - Fill back-pointers (`pcm`, `pstr`), `number = idx`, `stream`, `name = "subdevice #idx"`, `buffer_bytes_max = UINT_MAX`.
  - Link: if first, `pstr->substream = substream`; else `prev->next = substream`.
  - if !`pcm->internal`: `snd_pcm_substream_proc_init(substream)`. On err unlink+free, return.
  - `substream->group = &substream->self_group`; `snd_pcm_group_init(&substream->self_group)`.
  - `list_add_tail(&substream->link_list, &substream->self_group.substreams)`.
  - `atomic_set(&substream->mmap_count, 0)`.
- return 0.

REQ-11: `snd_pcm_dev_register(device) -> i32` (called by `snd_card_register` → `snd_device_register_all`):
- `pcm = device->device_data`.
- Lock `register_mutex`:
  - `snd_pcm_add(pcm)` — splice into `snd_pcm_devices` sorted by `(card->number, device)`; reject duplicates with `-EBUSY`.
  - For each direction `cidx ∈ {PLAYBACK, CAPTURE}`:
    - if `pstr->substream == NULL`: skip.
    - `devtype = SNDRV_DEVICE_TYPE_PCM_PLAYBACK` or `_PCM_CAPTURE`.
    - `snd_register_device(devtype, pcm->card, pcm->device, &snd_pcm_f_ops[cidx], pcm, pstr->dev)` — creates `/dev/snd/pcmCNDDp` or `c` minor. On err: `list_del_init(&pcm->list)`; return.
    - For each substream: `snd_pcm_timer_init(substream)`.
- `pcm_call_notify(pcm, n_register)` — invoke `snd_pcm_notify_list` callbacks.
- return 0.

REQ-12: `snd_pcm_dev_disconnect(device) -> i32`:
- Lock `register_mutex` AND `pcm->open_mutex`.
- `wake_up(&pcm->open_wait)` — abort blocked `snd_pcm_open` waiters.
- `list_del_init(&pcm->list)` — remove from global.
- `for_each_pcm_substream(pcm, cidx, substream)`:
  - `snd_pcm_stream_lock_irq(substream)`.
  - if `substream->runtime`:
    - if `snd_pcm_running(substream)`: `snd_pcm_stop(substream, SNDRV_PCM_STATE_DISCONNECTED)`.
    - `__snd_pcm_set_state(substream->runtime, SNDRV_PCM_STATE_DISCONNECTED)` (unconditional).
    - `wake_up(&runtime->sleep)`; `wake_up(&runtime->tsleep)`.
  - `snd_pcm_stream_unlock_irq(substream)`.
- `for_each_pcm_substream(pcm, cidx, substream)`: `snd_pcm_sync_stop(substream, false)`.
- `pcm_call_notify(pcm, n_disconnect)`.
- For each direction: if `pstr->dev`: `snd_unregister_device(pstr->dev)`; `free_chmap(pstr)`.
- return 0.

REQ-13: `snd_pcm_dev_free(device) -> i32`:
- `pcm = device->device_data`; return `snd_pcm_free(pcm)`.

REQ-14: `snd_pcm_free(pcm) -> i32`:
- if `pcm == NULL`: return 0.
- if !`pcm->internal`: `pcm_call_notify(pcm, n_unregister)`.
- if `pcm->private_free`: call.
- `snd_pcm_lib_preallocate_free_for_all(pcm)`.
- `snd_pcm_free_stream(&pcm->streams[PLAYBACK])`.
- `snd_pcm_free_stream(&pcm->streams[CAPTURE])`.
- `kfree(pcm)`.
- return 0.

REQ-15: `snd_pcm_free_stream(pstr)`:
- Walk `pstr->substream` linked list; for each substream: detach timer if any (`snd_pcm_timer_done`); free `substream->self_group` resources; `snd_pcm_substream_proc_done(substream)`; free runtime resources if still attached (shouldn't be at this point); `kfree(substream)`.
- `snd_pcm_stream_proc_done(pstr)`.
- `free_chmap(pstr)`.
- if `pstr->dev`: `put_device(pstr->dev)`.

REQ-16: `snd_pcm_attach_substream(pcm, stream, file, rsubstream) -> i32` (per-open hot path):
- Validate `pcm != NULL`, `rsubstream != NULL`, `stream ∈ {PLAYBACK, CAPTURE}`.
- `*rsubstream = NULL`.
- `pstr = &pcm->streams[stream]`.
- if `pstr->substream == NULL` ∨ `substream_count == 0`: return `-ENODEV`.
- `prefer_subdevice = snd_ctl_get_preferred_subdevice(card, SND_CTL_SUBDEV_PCM)`.
- HALF_DUPLEX guard: if `pcm->info_flags & HALF_DUPLEX`: scan opposite direction's substreams; if any `SUBSTREAM_BUSY`: return -EAGAIN.
- O_APPEND path (shared open):
  - if `prefer_subdevice < 0`: if `substream_count > 1` return -EINVAL; else `substream = pstr->substream`.
  - else: walk for `number == prefer_subdevice`.
  - if !substream: return -ENODEV.
  - if !SUBSTREAM_BUSY(substream): return -EBADFD.
  - `substream->ref_count++`; `*rsubstream = substream`; return 0.
- Normal open: walk `pstr->substream`; pick first !SUBSTREAM_BUSY ∧ (prefer == -1 ∨ number == prefer).
- if none: return -EAGAIN.
- Allocate `runtime = kzalloc(sizeof SndPcmRuntime)`. On NULL return -ENOMEM.
- `runtime->status = alloc_pages_exact(PAGE_ALIGN(sizeof SndPcmMmapStatus), GFP_KERNEL)`; zero it. On NULL: free runtime; return -ENOMEM.
- `runtime->control = alloc_pages_exact(PAGE_ALIGN(sizeof SndPcmMmapControl), GFP_KERNEL)`; zero it. On NULL: free status + runtime; return -ENOMEM.
- `init_waitqueue_head(&runtime->sleep)`; `init_waitqueue_head(&runtime->tsleep)`.
- `__snd_pcm_set_state(runtime, SNDRV_PCM_STATE_OPEN)`.
- `mutex_init(&runtime->buffer_mutex)`; `atomic_set(&runtime->buffer_accessing, 0)`.
- `substream->runtime = runtime`.
- `substream->private_data = pcm->private_data`.
- `substream->ref_count = 1`.
- `substream->f_flags = file->f_flags`.
- `substream->pid = get_pid(task_pid(current))`.
- `pstr->substream_opened++`.
- `*rsubstream = substream`.
- `#ifdef CONFIG_SND_PCM_XRUN_DEBUG`: `substream->xrun_counter = 0`.
- return 0.

REQ-17: `snd_pcm_detach_substream(substream)`:
- `PCM_RUNTIME_CHECK(substream)` returns if no runtime.
- `runtime = substream->runtime`.
- if `runtime->private_free`: call.
- `free_pages_exact(runtime->status, PAGE_ALIGN(sizeof SndPcmMmapStatus))`.
- `free_pages_exact(runtime->control, PAGE_ALIGN(sizeof SndPcmMmapControl))`.
- `kfree(runtime->hw_constraints.rules)`.
- if `substream->timer`: under `timer->lock`: `substream->runtime = NULL`. Else: `substream->runtime = NULL`.
- `mutex_destroy(&runtime->buffer_mutex)`.
- `snd_fasync_free(runtime->fasync)`.
- `kfree(runtime)`.
- `put_pid(substream->pid)`; `substream->pid = NULL`.
- `substream->pstr->substream_opened--`.

REQ-18: `snd_pcm_add(newpcm) -> i32`:
- Walk `snd_pcm_devices` in order; insert before first entry where `(pcm->card->number, pcm->device) > (newpcm->card->number, newpcm->device)`.
- Reject duplicates with -EBUSY.

REQ-19: `snd_pcm_get(card, device) -> *SndPcm`: walk `snd_pcm_devices` for `pcm->card == card ∧ pcm->device == device`.

REQ-20: `snd_pcm_next(card, device) -> i32`: return next device >= `device` for `card`, or -1.

REQ-21: `snd_pcm_control_ioctl(card, control, cmd, arg)` — handles per-card PCM enumeration ioctls on `/dev/snd/controlCN`:
- `SNDRV_CTL_IOCTL_PCM_NEXT_DEVICE` — userspace passes device index, returns next.
- `SNDRV_CTL_IOCTL_PCM_INFO` — fills `snd_pcm_info` for given (device, stream, subdevice).
- `SNDRV_CTL_IOCTL_PCM_PREFER_SUBDEVICE` — set preferred subdevice for next open by this control fd.

REQ-22: `snd_pcm_notify(notify, nfree)`:
- if `nfree == 0`: add to `snd_pcm_notify_list`; for every existing PCM call `notify->n_register`.
- if `nfree == 1`: for every existing PCM call `notify->n_disconnect` + `n_unregister`; remove from list.
- Used only by `CONFIG_SND_PCM_OSS`.

REQ-23: `/proc/asound/pcm` per-line format (`snd_pcm_proc_read`):
- `"NN-DD: shortname : longname : direction subdevices N/N\n"` per direction per PCM. Format-stable.

REQ-24: `/proc/asound/cardN/pcmDDx/info` (`snd_pcm_proc_info_read`):
- Lines: `card`, `device`, `subdevice`, `stream`, `id`, `name`, `subname`, `class`, `subclass`, `subdevices_count`, `subdevices_avail`.

REQ-25: `/proc/asound/cardN/pcmDDx/subN/hw_params` (`snd_pcm_substream_proc_hw_params_read`):
- Output `closed` if `runtime == NULL` else format/subformat/channels/rate/period_size/buffer_size/tick_time.

REQ-26: `/proc/asound/cardN/pcmDDx/subN/sw_params` (`snd_pcm_substream_proc_sw_params_read`):
- Output tstamp_mode, period_step, avail_min, start_threshold, stop_threshold, silence_threshold, silence_size, boundary.

REQ-27: `/proc/asound/cardN/pcmDDx/subN/status` (`snd_pcm_substream_proc_status_read`):
- state, owner_pid, trigger_time, tstamp, delay, avail, avail_max, hw_ptr, appl_ptr, suspended_state, audio_tstamp, driver_tstamp.

REQ-28: `/proc/asound/cardN/pcmDDx/subN/xrun_injection` (W): write to inject XRUN via `snd_pcm_stop_xrun`.

REQ-29: `/proc/asound/cardN/pcmDDx/subN/xrun_debug` (RW): per-`CONFIG_SND_PCM_XRUN_DEBUG`.

REQ-30: `snd_pcm_format_name(format) -> *u8`: returns NUL-terminated string from `snd_pcm_format_names[]`; NULL if unknown.

REQ-31: `pcm_class_show` sysfs attribute (`/sys/class/sound/pcmCNDDx/pcm_class`):
- Reads `pcm->dev_class`; emits `"generic" | "multi" | "modem" | "digitizer" | "none"`.

REQ-32: MMAP support contract (consumed by `pcm_native.c` `snd_pcm_mmap`):
- `runtime->status` page mmap'd at `SNDRV_PCM_MMAP_OFFSET_STATUS` — RO; userspace polls `state`, `hw_ptr`, `tstamp`.
- `runtime->control` page mmap'd at `SNDRV_PCM_MMAP_OFFSET_CONTROL` — RW; userspace writes `appl_ptr`, `avail_min`.
- `runtime->dma_area` mmap'd at offset 0 — RW; size `runtime->dma_bytes`; refcounted by `mmap_count`.
- Page-fault handler / `vm_ops` from driver `snd_pcm_ops.mmap` (default: `snd_pcm_default_mmap`).

REQ-33: Buffer prealloc / managed:
- Legacy: `snd_pcm_lib_preallocate_pages_for_all(pcm, type, data, size, max)` — prealloc per substream; driver `hw_params` then uses `snd_pcm_lib_malloc_pages`.
- Preferred: `snd_pcm_set_managed_buffer_all(pcm, type, data, size, max)` — ALSA core manages alloc/free across hw_params/hw_free transitions; driver does not implement them.

REQ-34: `register_mutex` scope:
- Serializes: `snd_pcm_dev_register`, `snd_pcm_dev_disconnect`, `snd_pcm_notify` add/remove, `snd_pcm_add`.
- Ordering: `register_mutex` before `pcm->open_mutex` before `runtime->buffer_mutex` before substream stream-lock.

## Acceptance Criteria

- [ ] AC-1: `snd_pcm_new(card, "id", 0, 1, 1, &pcm)` returns 0; `pcm->streams[0].substream_count == 1` ∧ `pcm->streams[1].substream_count == 1`; a `snd_device` of type `SNDRV_DEV_PCM` attached to card.
- [ ] AC-2: After `snd_card_register`, `/dev/snd/pcmC0D0p` and `/dev/snd/pcmC0D0c` exist; `open()` blocks no waiter (substream available).
- [ ] AC-3: `snd_pcm_new_internal` produces no `/dev/snd/pcm*` and no `/proc/asound/.../pcm*`.
- [ ] AC-4: `snd_pcm_attach_substream` on busy substream returns `-EAGAIN`; with `O_APPEND` and existing busy substream returns 0 and increments `ref_count`.
- [ ] AC-5: `snd_pcm_attach_substream` on HALF_DUPLEX PCM with opposite direction busy returns `-EAGAIN`.
- [ ] AC-6: After attach: `runtime->state == SNDRV_PCM_STATE_OPEN` ∧ `runtime->status->state == OPEN` ∧ `runtime->status` page mmap-able at `SNDRV_PCM_MMAP_OFFSET_STATUS` ∧ `runtime->control` mmap-able at `SNDRV_PCM_MMAP_OFFSET_CONTROL`.
- [ ] AC-7: `__snd_pcm_set_state(runtime, S)` mutates both `runtime->state` and `runtime->status->state` atomically from a stream-lock holder's POV.
- [ ] AC-8: `snd_pcm_detach_substream` frees status, control, runtime; `substream->runtime == NULL` ∧ `pstr->substream_opened` decremented.
- [ ] AC-9: `snd_pcm_dev_disconnect` while any substream RUNNING: `snd_pcm_stop` invoked then `state = DISCONNECTED`; `wake_up` fires on `runtime->sleep` and `runtime->tsleep`.
- [ ] AC-10: Post-disconnect, every state-transition ioctl on the substream returns `-ENODEV` ∨ `-EBADFD`; only `close` succeeds.
- [ ] AC-11: `snd_pcm_dev_register` enforces `(card,device)` uniqueness via `snd_pcm_add`; collision returns `-EBUSY`.
- [ ] AC-12: `/proc/asound/cardN/pcmDDx/info` and `subN/{hw_params,sw_params,status}` exist for every non-internal PCM after register; absent before; removed on disconnect.
- [ ] AC-13: `snd_pcm_control_ioctl(SNDRV_CTL_IOCTL_PCM_NEXT_DEVICE)` walks `pstr->substream_count` in sorted device order.
- [ ] AC-14: `snd_pcm_notify(notify, 0)` invokes `notify->n_register(pcm)` for every currently-registered PCM exactly once.
- [ ] AC-15: `snd_pcm_free` releases preallocated buffers (`snd_pcm_lib_preallocate_free_for_all`) before freeing streams.

## Architecture

```
struct SndPcm {
  card: *SndCard,
  device: i32,
  id: [u8; 64],
  streams: [SndPcmStr; 2],                 // 0 = PLAYBACK, 1 = CAPTURE
  info_flags: u32,
  dev_class: u32,
  dev_subclass: u32,
  internal: bool,
  nonatomic: bool,
  no_device_suspend: bool,
  open_mutex: Mutex,
  open_wait: WaitQueueHead,
  list: ListHead,                          // in PCM::DEVICES
  private_data: *void,
  private_free: Option<fn(*SndPcm)>,
}

struct SndPcmStr {
  stream: i32,
  substream_count: u32,
  substream_opened: u32,
  substream: *SndPcmSubstream,             // head of next-linked list
  chmap_kctl: *SndKctl,
  dev: *Device,                            // /sys/class/sound/pcmCNDDx
  pcm: *SndPcm,
  proc_root: *SndInfoEntry,
  oss: SndPcmOssStream,                    // CONFIG_SND_PCM_OSS
}

struct SndPcmSubstream {
  pcm: *SndPcm,
  pstr: *SndPcmStr,
  number: u32,
  stream: i32,
  name: [u8; 32],
  next: *SndPcmSubstream,
  ops: *SndPcmOps,
  runtime: *SndPcmRuntime,                 // non-NULL between attach/detach
  private_data: *void,
  ref_count: i32,
  f_flags: u32,
  pid: *Pid,
  group: *SndPcmGroup,
  self_group: SndPcmGroup,
  link_list: ListHead,
  mmap_count: AtomicI32,
  buffer_bytes_max: u32,
  pcm_release: Option<fn(*SndPcmSubstream)>,
  timer: *SndTimer,
  xrun_counter: u32,                       // CONFIG_SND_PCM_XRUN_DEBUG
}

struct SndPcmRuntime {
  state: SndPcmStateT,
  suspended_state: SndPcmStateT,
  status: *SndPcmMmapStatus,               // alloc_pages_exact, mmap-RO
  control: *SndPcmMmapControl,             // alloc_pages_exact, mmap-RW
  sleep: WaitQueueHead,
  tsleep: WaitQueueHead,
  buffer_mutex: Mutex,
  buffer_accessing: AtomicI32,
  hw: SndPcmHardware,
  hw_constraints: SndPcmHwConstraints,
  format: SndPcmFormatT,
  access: SndPcmAccessT,
  subformat: SndPcmSubformatT,
  rate: u32,
  channels: u32,
  period_size: u64,
  period_count: u32,
  buffer_size: u64,
  frame_bits: u32,
  sample_bits: u32,
  byte_align: u32,
  dma_buffer_p: *SndDmaBuffer,
  dma_area: *void,
  dma_addr: dma_addr_t,
  dma_bytes: usize,
  xrun_threshold: u64,
  stop_threshold: u64,
  silence_threshold: u64,
  silence_size: u64,
  boundary: u32,
  fasync: *SndFasync,
  private_data: *void,
  private_free: Option<fn(*SndPcmRuntime)>,
  oss: SndPcmOssRuntime,                   // CONFIG_SND_PCM_OSS
}

struct SndPcmOps {
  open: Option<fn(*SndPcmSubstream) -> i32>,
  close: Option<fn(*SndPcmSubstream) -> i32>,
  ioctl: Option<fn(*SndPcmSubstream, u32, *void) -> i32>,
  hw_params: Option<fn(*SndPcmSubstream, *SndPcmHwParams) -> i32>,
  hw_free: Option<fn(*SndPcmSubstream) -> i32>,
  prepare: Option<fn(*SndPcmSubstream) -> i32>,
  trigger: Option<fn(*SndPcmSubstream, i32) -> i32>,
  sync_stop: Option<fn(*SndPcmSubstream) -> i32>,
  pointer: Option<fn(*SndPcmSubstream) -> SndPcmUframesT>,
  get_time_info: Option<fn(...)>,
  fill_silence: Option<fn(...) -> i32>,
  copy: Option<fn(...) -> i32>,
  page: Option<fn(*SndPcmSubstream, u64) -> *Page>,
  mmap: Option<fn(*SndPcmSubstream, *VmAreaStruct) -> i32>,
  ack: Option<fn(*SndPcmSubstream) -> i32>,
}

static DEVICES: ListHead<SndPcm>;            // global, sorted (card.number, device)
static NOTIFY_LIST: ListHead<SndPcmNotify>;
static REGISTER_MUTEX: Mutex;
```

`SndPcm::new(card, id, device, playback_count, capture_count) -> Result<*SndPcm, Errno>`:
1. Delegate to `SndPcm::new_inner(card, id, device, playback_count, capture_count, internal=false)`.

`SndPcm::new_internal(...)`: same but `internal=true`.

`SndPcm::new_inner(card, id, device, playback_count, capture_count, internal) -> Result<*SndPcm, Errno>`:
1. Define `OPS` (full vtable) and `INTERNAL_OPS` (only `dev_free`).
2. `pcm = kzalloc(sizeof SndPcm, GFP_KERNEL)` or return -ENOMEM.
3. Init fields; `pcm.card = card`; `pcm.device = device`; `pcm.internal = internal`.
4. Init `open_mutex`, `open_wait`, `list`.
5. if id: strscpy.
6. `SndPcm::new_stream(pcm, PLAYBACK, playback_count)`. On err goto free.
7. `SndPcm::new_stream(pcm, CAPTURE, capture_count)`. On err goto free.
8. `SndDevice::new(card, SNDRV_DEV_PCM, pcm, if internal then INTERNAL_OPS else OPS)`. On err goto free.
9. Return Ok(pcm).
10. free: `SndPcm::free(pcm)`; return err.

`SndPcm::new_stream(pcm, stream, substream_count) -> Result<(), Errno>`:
1. `pstr = &pcm.streams[stream]`.
2. `pstr.stream = stream`; `pstr.pcm = pcm`; `pstr.substream_count = substream_count`.
3. if substream_count == 0: return Ok.
4. `snd_device_alloc(&pstr.dev, pcm.card)`. On err return.
5. `dev_set_name(pstr.dev, "pcmC{N}D{D}{p|c}")`.
6. Set `pstr.dev.groups`, `pstr.dev.type`, `dev_set_drvdata`.
7. if !pcm.internal: `snd_pcm_stream_proc_init(pstr)`. On err return.
8. Build singly-linked substream chain for idx in 0..substream_count:
   - Allocate substream; populate back-pointers + name.
   - Splice into list (head → prev.next).
   - if !pcm.internal: `snd_pcm_substream_proc_init`. On err unlink/free/return.
   - Init self_group + link_list; mmap_count = 0.
9. Return Ok.

`SndPcmSubstream::attach(pcm, stream, file) -> Result<*SndPcmSubstream, Errno>`:
1. Validate inputs.
2. `pstr = &pcm.streams[stream]`; bail with -ENODEV if empty.
3. `prefer = snd_ctl_get_preferred_subdevice(card, SND_CTL_SUBDEV_PCM)`.
4. HALF_DUPLEX: if any opposite-direction substream BUSY, return -EAGAIN.
5. O_APPEND branch:
   - Resolve substream by prefer or singleton constraint.
   - if !BUSY: return -EBADFD.
   - substream.ref_count += 1; return Ok(substream).
6. Normal branch:
   - Walk pstr.substream for !BUSY ∧ (prefer == -1 ∨ number == prefer).
   - if none: return -EAGAIN.
7. Allocate runtime (kzalloc); status (alloc_pages_exact, page-aligned); control (alloc_pages_exact). On any failure: cascade-free; return -ENOMEM.
8. Init runtime.sleep, runtime.tsleep, runtime.buffer_mutex, runtime.buffer_accessing.
9. `SndPcmRuntime::set_state(runtime, SNDRV_PCM_STATE_OPEN)`.
10. Wire substream.runtime, private_data, ref_count = 1, f_flags = file.f_flags, pid = get_pid(current).
11. pstr.substream_opened += 1.
12. Return Ok(substream).

`SndPcmSubstream::detach(substream)`:
1. PCM_RUNTIME_CHECK guard.
2. runtime = substream.runtime.
3. runtime.private_free?(runtime).
4. free_pages_exact(runtime.status, PAGE_ALIGN(sizeof SndPcmMmapStatus)).
5. free_pages_exact(runtime.control, PAGE_ALIGN(sizeof SndPcmMmapControl)).
6. kfree(runtime.hw_constraints.rules).
7. Detach from timer if any (under timer.lock); else direct.
8. mutex_destroy(buffer_mutex); snd_fasync_free; kfree(runtime).
9. put_pid(substream.pid); substream.pid = NULL.
10. pstr.substream_opened -= 1.

`SndPcm::dev_register(device) -> Result<(), Errno>`:
1. pcm = device.device_data.
2. Lock REGISTER_MUTEX:
   - `SndPcm::add_to_list(pcm)` (sorted; reject duplicates -EBUSY).
   - For cidx ∈ {0, 1}:
     - if !pstr.substream: continue.
     - devtype = if cidx == 0 PCM_PLAYBACK else PCM_CAPTURE.
     - `snd_register_device(devtype, card, device, &snd_pcm_f_ops[cidx], pcm, pstr.dev)`. On err: unlink + return.
     - For substream in pstr: `snd_pcm_timer_init(substream)`.
3. pcm_call_notify(pcm, n_register).
4. Return Ok.

`SndPcm::dev_disconnect(device) -> Result<(), Errno>`:
1. Lock REGISTER_MUTEX; lock pcm.open_mutex.
2. wake_up(&pcm.open_wait).
3. list_del_init(&pcm.list).
4. for_each_pcm_substream(pcm, cidx, substream):
   - stream_lock_irq.
   - if substream.runtime:
     - if running: `SndPcmSubstream::stop(substream, DISCONNECTED)`.
     - `SndPcmRuntime::set_state(runtime, DISCONNECTED)`.
     - wake_up sleep, tsleep.
   - stream_unlock_irq.
5. for_each_pcm_substream(pcm, cidx, substream): `SndPcmSubstream::sync_stop(substream, sync_irq=false)`.
6. pcm_call_notify(pcm, n_disconnect).
7. For cidx ∈ {0,1}: if pstr.dev: snd_unregister_device(pstr.dev); free_chmap(pstr).
8. Return Ok.

`SndPcm::dev_free(device) -> Result<(), Errno>`:
1. Return `SndPcm::free(device.device_data)`.

`SndPcm::free(pcm) -> Result<(), Errno>`:
1. if pcm.is_null(): return Ok.
2. if !pcm.internal: pcm_call_notify(pcm, n_unregister).
3. pcm.private_free?(pcm).
4. `SndPcmMem::preallocate_free_for_all(pcm)`.
5. `SndPcm::free_stream(&pcm.streams[0])`.
6. `SndPcm::free_stream(&pcm.streams[1])`.
7. kfree(pcm).
8. Return Ok.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `substream_state_init_open` | INVARIANT | post-`attach`: runtime.state == OPEN ∧ runtime.status.state == OPEN. |
| `status_control_pages_page_aligned` | INVARIANT | runtime.status and runtime.control are page-aligned and exactly `PAGE_ALIGN(sizeof)` bytes from `alloc_pages_exact`. |
| `attach_failure_no_leak` | INVARIANT | any branch of attach that returns Err: no runtime / status / control / pid allocation outlives the call. |
| `detach_clears_runtime` | INVARIANT | post-detach: substream.runtime == NULL ∧ substream.pid == NULL. |
| `substream_count_invariant` | INVARIANT | pstr.substream_opened == count(substream where SUBSTREAM_BUSY). |
| `disconnect_idempotent` | INVARIANT | dev_disconnect called twice: second call no-op (list_del_init handles re-entry). |
| `state_disconnected_terminal` | INVARIANT | once state == DISCONNECTED, only detach is valid; all ioctl entries reject. |
| `register_then_disconnect_pairs_unique` | INVARIANT | (card.number, device) uniqueness preserved across snd_pcm_add lifecycle. |
| `half_duplex_mutex` | INVARIANT | per-HALF_DUPLEX: PLAYBACK busy ⊕ CAPTURE busy (mutually exclusive). |

### Layer 2: TLA+

`sound/core/pcm-lifecycle.tla` covers per-PCM lifecycle and per-substream state machine. Actions:
- `New` (allocate pcm, new_stream P, new_stream C, snd_device_new).
- `Register` (snd_pcm_add, snd_register_device per direction, timer_init).
- `Open[s]` (attach_substream: state → OPEN, substream_opened++).
- `HwParams[s]` (OPEN/SETUP → SETUP).
- `Prepare[s]` (SETUP/RUNNING/XRUN → PREPARED).
- `Trigger[s, START]` (PREPARED → RUNNING).
- `Trigger[s, PAUSE_PUSH]` (RUNNING → PAUSED).
- `Trigger[s, PAUSE_RELEASE]` (PAUSED → RUNNING).
- `Trigger[s, STOP]` (RUNNING/PAUSED → SETUP via prepare or implicit).
- `Drain[s]` (RUNNING → DRAINING → SETUP).
- `Xrun[s]` (RUNNING → XRUN).
- `Suspend[s]` (any → SUSPENDED; save suspended_state).
- `Resume[s]` (SUSPENDED → suspended_state).
- `Close[s]` (any → "detached").
- `Disconnect` (any → DISCONNECTED for every s; remove from DEVICES; unregister minors).

Properties:
- `safety_state_transitions_legal` — every state change matches an action's pre/post.
- `safety_open_then_close_balanced` — each Open[s] eventually paired with Close[s] (live closes; bounded if no leak).
- `safety_no_running_after_disconnect` — state == DISCONNECTED ⟹ no future Trigger.
- `safety_status_state_mirror` — every set_state writes both runtime.state and runtime.status.state.
- `safety_half_duplex_mutex` — HALF_DUPLEX ⟹ ¬(PLAYBACK busy ∧ CAPTURE busy).
- `liveness_disconnect_terminates` — Disconnect completes in finite steps.
- `liveness_drain_completes_or_xruns` — DRAINING eventually reaches SETUP or XRUN.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SndPcm::new_inner` post: 2 streams populated; SndDevice attached; on err no leak | `SndPcm::new_inner` |
| `SndPcm::new_stream` post: substream_count substreams linked via .next; self_group per substream | `SndPcm::new_stream` |
| `SndPcmSubstream::attach` post: runtime.state == OPEN ∧ status/control page-allocated ∧ ref_count == 1 (or ++ for O_APPEND) | `SndPcmSubstream::attach` |
| `SndPcmSubstream::detach` post: runtime == NULL; status/control freed; substream_opened-- | `SndPcmSubstream::detach` |
| `SndPcm::dev_register` post: pcm in DEVICES at sorted position; both minor devices registered | `SndPcm::dev_register` |
| `SndPcm::dev_disconnect` post: pcm not in DEVICES; all runtimes state == DISCONNECTED; minors unregistered | `SndPcm::dev_disconnect` |
| `SndPcm::free` post: prealloc freed before stream free; kfree(pcm) last | `SndPcm::free` |
| `SndPcmRuntime::set_state` post: runtime.state == s ∧ runtime.status.state == s | `SndPcmRuntime::set_state` |

### Layer 4: Verus/Creusot functional

End-to-end equivalence verified by alsa-lib regression suite:
- `aplay -D hw:0,0 <wav>` (PLAYBACK lifecycle: OPEN → hw_params → prepare → trigger START → RUNNING → write → drain → close).
- `arecord -D hw:0,0 -d 5 <wav>` (CAPTURE lifecycle parity).
- `speaker-test --pause` (PAUSE_PUSH / PAUSE_RELEASE round-trip).
- `aplay <large.wav>` interrupted by USB-audio surprise removal: expect EPOLLERR + write returns -ENODEV.
- mmap mode (`snd_pcm_mmap_begin` / `_commit`): status/control pages mmapped at canonical offsets, hw_ptr/appl_ptr updates observable.
- `/proc/asound/pcm`, `/proc/asound/cardN/pcmDDx/info|status|hw_params|sw_params` text format byte-equivalent to upstream.

## Hardening

(Inherits row-1 features from `sound/00-overview.md` § Hardening.)

ALSA PCM reinforcement:

- **Per-`pstr->substream` walked under `pcm->open_mutex`** — defense against per-double-open race on the same subdevice.
- **Per-`SUBSTREAM_BUSY` arbitration** — defense against per-stale-runtime in concurrent attach.
- **Per-`alloc_pages_exact` for status/control** — defense against per-mmap-leaks-kheap-pointer (page granularity for user mmap).
- **Per-`__snd_pcm_set_state` writes both fields** — defense against per-userspace-reads-stale-mmap-state divergence.
- **Per-`runtime->buffer_mutex` + `buffer_accessing`** — defense against per-hw_params-mid-DMA UAF.
- **Per-`SNDRV_PCM_STATE_DISCONNECTED` terminal** — defense against per-hot-unplug ioctl on freed driver state.
- **Per-`snd_pcm_stream_lock_irq` around state read+act** — defense against per-state-check-TOCTOU.
- **Per-`HALF_DUPLEX` mutual exclusion** — defense against per-modem-PCM concurrent P+C.
- **Per-`(card,device)` uniqueness via `snd_pcm_add`** — defense against per-double-register on same minor.
- **Per-`register_mutex` ordering** — defense against per-register-vs-disconnect race.
- **Per-`get_pid` / `put_pid`** — defense against per-pid-namespace UAF on substream owner.
- **Per-`mmap_count` ref** — defense against per-munmap-while-DMA-active.
- **Per-disconnect wake on `runtime->sleep` ∧ `runtime->tsleep`** — defense against per-read/write blocked-forever after unplug.
- **Per-internal PCM no userspace surface** — defense against per-ASoC-BE-PCM accidentally exposed via `/dev/snd/*`.
- **Per-`pcm_call_notify` only for `!internal`** — defense against per-OSS-emul-confused-by-BE-PCM.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — `snd_pcm_hw_params`, `snd_pcm_sw_params`, `snd_pcm_channel_info`, `snd_pcm_sync_ptr`, and `snd_xferi/xfern` payloads bounded against fixed struct sizes; per-channel `area` arrays length-checked against `runtime->channels`.
- **PAX_KERNEXEC** — `snd_pcm_f_ops`, ioctl dispatcher, and `snd_pcm_ops` `open/close/hw_params/prepare/trigger/pointer/copy_user/mmap` callbacks reside in RX `.text`; per-driver ops registered through kCFI-signed indirection.
- **PAX_RANDKSTACK** — kstack-offset randomization on every `/dev/snd/pcmC*D*[cp]` ioctl / read / write / mmap entry.
- **PAX_REFCOUNT** — saturating refcount on `snd_pcm_substream`, `snd_pcm_oss_file`, and the parent `snd_pcm`; defense against draining-vs-disconnect races wrapping the count.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `snd_pcm_runtime` (contains DMA mappings, status/control mmap pages) and per-substream DMA buffers on `snd_pcm_release` (defense against next-open inheriting prior audio data).
- **PAX_UDEREF (SMAP/SMEP)** — ASM_CLAC on every PCM ioctl and `copy_user` from user-space; `snd_pcm_lib_write/read` validates `frames` against `runtime->buffer_size`.
- **PAX_RAP / kCFI** — driver-supplied `snd_pcm_ops` and `snd_pcm_dma_buffer.ops` dispatched via kCFI-signed indirect calls.
- **GRKERNSEC_HIDESYM** — `snd_pcm_devices`, `snd_pcm_link_rwsem`, and per-substream `runtime` symbols masked from /proc/kallsyms for non-CAP_SYSLOG.
- **GRKERNSEC_DMESG** — PCM xrun / hw_params-mismatch / disconnect diagnostics restricted to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict on /dev/snd/pcm*** — devnode access governed by audio group; CAP_SYS_ADMIN required for udev rule override.
- **`snd_pcm_link_rwsem` strict-write** — defense against per-disconnect linker writing while reader walks linked-substream list.
- **`pcm_call_notify` only for `!internal`** — defense against OSS-emul confused by ASoC backend PCM accidentally exposed via /dev/snd/*.
- **Disconnect wake on `runtime->sleep` ∧ `runtime->tsleep`** — defense against read/write blocked-forever after unplug.

Per-doc rationale: `/dev/snd/pcm*` is the streaming-data path between userspace and audio DSP — `runtime` carries DMA-coherent buffers mmaped read-write into userspace, and `status`/`control` pages are publicly mmap'd. UAF here is directly weaponizable as ringbuffer-pointer corruption + DMA-into-arbitrary-RAM. Triple-overlap (USERCOPY + REFCOUNT + MEMORY_SANITIZE) plus disconnect-wake on every blocking primitive is non-negotiable.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `sound/core/pcm_native.c` — `snd_pcm_open`, `_close`, `_hw_params`, `_sw_params`, `_prepare`, `_trigger`, `_drain`, `_mmap`, `_drop`, `_rewind`, `_forward`, ioctl dispatcher (covered separately in `pcm-native.md` Tier-3 if expanded)
- `sound/core/pcm_lib.c` — `snd_pcm_lib_malloc_pages`, `_write`, `_read`, hw_constraints helpers (covered separately)
- `sound/core/pcm_memory.c` — DMA preallocation internals and `snd_pcm_set_managed_buffer*` (covered separately)
- `sound/core/pcm_dmaengine.c` — generic dmaengine PCM (covered separately)
- `sound/core/pcm_compat.c` — 32-on-64 compat ioctls (covered separately)
- `sound/core/pcm_misc.c` — format tables (consumed here only via `snd_pcm_format_name`)
- `sound/core/pcm_timer.c` — `snd_pcm_timer_init` (covered in `timer.md` Tier-3 if expanded)
- `sound/core/oss/` — OSS PCM emulation (covered separately if needed)
- Driver-side `snd_pcm_ops` implementations (per-bus, e.g. HD-Audio, USB-audio)
- Implementation code
