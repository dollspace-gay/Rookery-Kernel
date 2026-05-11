# Tier-3: sound/core/rawmidi.c — ALSA raw MIDI (legacy + UMP) substreams

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: sound/00-overview.md
upstream-paths:
  - sound/core/rawmidi.c (~2136 lines)
  - sound/core/rawmidi_compat.c (32-bit compat ioctls)
  - include/sound/rawmidi.h (struct snd_rawmidi, snd_rawmidi_substream, snd_rawmidi_str, snd_rawmidi_runtime, snd_rawmidi_ops, snd_rawmidi_file, SNDRV_RAWMIDI_LFLG_*)
  - include/uapi/sound/asound.h (struct snd_rawmidi_info, snd_rawmidi_params, snd_rawmidi_status, snd_rawmidi_framing_tstamp, SNDRV_RAWMIDI_IOCTL_*, SNDRV_RAWMIDI_INFO_*, SNDRV_RAWMIDI_MODE_*)
  - include/sound/ump.h (Universal MIDI Packet 2.0 endpoint/block info)
-->

## Summary

ALSA's **raw MIDI** API provides a byte-stream channel to/from a hardware MIDI port (legacy MIDI 1.0 over UART or USB-MIDI 1.0) or a **Universal MIDI Packet (UMP)** endpoint (MIDI 2.0). Per `/dev/snd/midiC<card>D<dev>` per-device character device (or `umpC<card>D<dev>` for UMP endpoints). Per-direction: each `struct snd_rawmidi` carries two `struct snd_rawmidi_str` (INPUT, OUTPUT), each with a list of `struct snd_rawmidi_substream` (one per hardware subdevice, e.g. each MIDI port on a multi-port box). Per-substream runtime: `struct snd_rawmidi_runtime` owns a kvzalloc'd ring buffer (`buffer`, `buffer_size`, `hw_ptr` for kernel/hw side, `appl_ptr` for userspace side, `avail` counter, `avail_min` watermark). Per-direction IO model: blocking by default with 30s timeouts on write, NONBLOCK + O_APPEND optional, O_DSYNC drains output before return. Per-ioctl param control: PARAMS sets buffer_size/avail_min/no_active_sensing/framing/clock_type, STATUS reports avail/xruns, DROP discards pending, DRAIN waits for empty. Per-tstamp framing: 32-byte `snd_rawmidi_framing_tstamp` records (16 B data + tv_sec/tv_nsec) for accurate input timestamping. Per-UMP: 32-bit MIDI 2.0 packets, 4-byte alignment enforced, exposed via SNDRV_RAWMIDI_INFO_UMP and `umpC*D*` device nodes. Critical for: hardware MIDI port access (DAWs, synths), USB-MIDI 1.0 class drivers, UMP / MIDI 2.0 endpoints.

This Tier-3 covers `sound/core/rawmidi.c` (~2136 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct snd_rawmidi` | per-device (card+device) container | `RawMidi` |
| `struct snd_rawmidi_str` | per-direction (in/out) substream list | `RawMidiStr` |
| `struct snd_rawmidi_substream` | per-port substream state | `RawMidiSubstream` |
| `struct snd_rawmidi_runtime` | per-substream ring-buffer runtime | `RawMidiRuntime` |
| `struct snd_rawmidi_ops` | per-driver ops (open/close/trigger/drain) | `RawMidiOps` |
| `struct snd_rawmidi_file` | per-open-fd substream pair (input, output) | `RawMidiFile` |
| `struct snd_rawmidi_params` | per-ioctl param block (buffer_size, avail_min, mode, no_active_sensing) | `RawMidiParams` |
| `struct snd_rawmidi_status64` | per-ioctl status (stream, tstamp, avail, xruns) | `RawMidiStatus` |
| `struct snd_rawmidi_info` | per-ioctl info (card, device, subdevice, name, flags) | `RawMidiInfo` |
| `struct snd_rawmidi_framing_tstamp` | per-input-frame tstamp + 16 B data | `RawMidiFramingTstamp` |
| `snd_rawmidi_new()` | per-allocate + init | `RawMidi::new` |
| `snd_rawmidi_init()` | per-init (legacy + UMP shared path) | `RawMidi::init` |
| `snd_rawmidi_free()` | per-tear-down | `RawMidi::free` |
| `snd_rawmidi_set_ops()` | per-direction ops install | `RawMidi::set_ops` |
| `snd_rawmidi_alloc_substreams()` | per-direction substream allocation | `RawMidi::alloc_substreams` |
| `snd_rawmidi_open()` | per-userspace open (legacy + OSS minor multiplex) | `RawMidi::open` |
| `snd_rawmidi_release()` | per-userspace close | `RawMidi::release` |
| `snd_rawmidi_kernel_open()` / `_release()` | per-in-kernel client (e.g. seq_midi.c) open/close | `RawMidi::kernel_open` |
| `rawmidi_open_priv()` / `assign_substream()` / `open_substream()` / `close_substream()` | per-direction substream attach/detach | `RawMidi::open_priv` |
| `snd_rawmidi_runtime_create()` / `_free()` | per-substream ring buffer alloc / free | `RawMidi::runtime_create` |
| `snd_rawmidi_ioctl()` | per-fd ioctl mux | `RawMidi::ioctl` |
| `snd_rawmidi_control_ioctl()` | per-/dev/snd/controlC ioctl extension (RAWMIDI_*, UMP_*) | `RawMidi::control_ioctl` |
| `snd_rawmidi_info()` / `_info_select()` | per-substream info populate | `RawMidi::info` |
| `snd_rawmidi_output_params()` / `_input_params()` / `resize_runtime_buffer()` | per-direction param set + ring resize | `RawMidi::output_params` |
| `snd_rawmidi_output_status()` / `_input_status()` | per-direction status snapshot | `RawMidi::output_status` |
| `snd_rawmidi_drop_output()` / `_drain_output()` / `_drain_input()` | per-direction discard / wait-empty | `RawMidi::drop_output` |
| `snd_rawmidi_read()` / `snd_rawmidi_write()` | per-fd ring IO with wait | `RawMidi::read` / `write` |
| `snd_rawmidi_kernel_read()` / `_kernel_write()` | per-kernel-client ring IO | `RawMidi::kernel_read` |
| `snd_rawmidi_receive()` | per-hardware-irq-callback enqueue input | `RawMidi::receive` |
| `snd_rawmidi_transmit_peek()` / `_transmit_ack()` / `_transmit()` | per-driver-tx output ring drain | `RawMidi::transmit_peek` |
| `snd_rawmidi_transmit_empty()` | per-output empty-check | `RawMidi::transmit_empty` |
| `snd_rawmidi_proceed()` | per-driver discard-pending after error | `RawMidi::proceed` |
| `snd_rawmidi_poll()` | per-fd EPOLLIN/EPOLLOUT poll | `RawMidi::poll` |
| `receive_with_tstamp_framing()` / `get_framing_tstamp()` | per-input tstamp-framing path | `RawMidi::receive_framed` |
| `snd_rawmidi_buffer_ref()` / `_unref()` / `_ref_sync()` | per-runtime buffer pin count for resize-safety | `RawMidi::buffer_ref` |
| `snd_rawmidi_input_trigger()` / `_output_trigger()` | per-direction hw trigger gate | `RawMidi::input_trigger` |
| `snd_rawmidi_dev_register()` / `_disconnect()` / `_free()` | per-snd_device callbacks | `RawMidi::dev_register` |
| `snd_rawmidi_next_device()` / `snd_rawmidi_call_ump_ioctl()` | per-control-fd enumeration of MIDI / UMP devices | `RawMidi::next_device` |
| `snd_rawmidi_proc_info_read()` | per-/proc/asound/cardN/midiD info | `RawMidi::proc_info_read` |

## Compatibility contract

REQ-1: struct snd_rawmidi:
- card: per-snd_card backreference.
- device: index in card.
- streams[2]: per-direction RawMidiStr (INPUT=0, OUTPUT=1).
- info_flags: bitmask of SNDRV_RAWMIDI_INFO_OUTPUT | _INPUT | _DUPLEX | _UMP | _STREAM_INACTIVE.
- id[64], name[80]: identifiers.
- open_mutex: serializes open/close.
- open_wait: blocks for busy substream.
- ops: optional rmidi-level ops (ioctl, dev_register, dev_unregister, proc_read).
- list: linkage on snd_rawmidi_devices (global, register_mutex-protected).
- private_data, private_free.
- proc_entry, dev, seq_dev, tied_device.

REQ-2: struct snd_rawmidi_str:
- substreams: list of snd_rawmidi_substream (typically 1 per port).
- substream_count: total.
- substream_opened: currently in-use count.
- dev_attr_*: sysfs.

REQ-3: struct snd_rawmidi_substream:
- list, number (subdevice index), stream (INPUT|OUTPUT), name[32].
- rmidi backref, pstr (RawMidiStr) backref.
- runtime: kvzalloc'd RawMidiRuntime when opened.
- ops: per-direction RawMidiOps.
- lock: spinlock guarding runtime ring state.
- use_count: open-count (for append-mode multiplexing).
- opened, append, active_sensing, framing, clock_type, inactive flags.
- pid: get_pid of opener.
- bytes: lifetime byte counter (Tx/Rx).

REQ-4: struct snd_rawmidi_runtime:
- substream backref.
- buffer: kvzalloc, default size PAGE_SIZE.
- buffer_size: current.
- appl_ptr: userspace index.
- hw_ptr: kernel/hw index.
- avail: bytes consumable in current direction (= avail-to-read for INPUT, = free-space for OUTPUT).
- avail_min: poll watermark (default 1).
- xruns: cumulative input overrun bytes (zeroed on each status snapshot).
- drain: 1 while drain_output is waiting.
- buffer_ref: pin count blocking resize while a reader/writer is mid-loop.
- align: 0 (legacy) or 3 (UMP — 4-byte packet alignment).
- event: optional kernel-callback on input.
- event_work: workqueue item for event delivery.
- sleep: waitqueue for read/write/drain.
- oss: 1 if opened via /dev/midi*<n> legacy OSS minor.
- private_data / private_free.

REQ-5: struct snd_rawmidi_ops (per-direction driver callbacks):
- open(substream): hardware activate.
- close(substream): hardware deactivate.
- trigger(substream, up): start/stop transfer (interrupt mode).
- drain(substream): block until tx FIFO empty (optional; else 50ms msleep fallback).

REQ-6: Open path: snd_rawmidi_open(inode, file):
- (O_APPEND && !O_NONBLOCK) ⟹ -EINVAL.
- stream_open(inode, file).
- maj = imajor(inode). If maj == snd_major: snd_lookup_minor_data(SNDRV_DEVICE_TYPE_RAWMIDI). Else if maj == SOUND_MAJOR (CONFIG_SND_OSSEMUL): snd_lookup_oss_minor_data(SNDRV_OSS_DEVICE_TYPE_MIDI). Else -ENXIO.
- try_module_get(rmidi.card.module).
- mutex_lock(rmidi.open_mutex).
- snd_card_file_add(card, file).
- fflags = snd_rawmidi_file_flags(file): FMODE_WRITE-only ⟹ LFLG_OUTPUT; FMODE_READ-only ⟹ LFLG_INPUT; both ⟹ LFLG_OPEN (= OUTPUT|INPUT).
- O_APPEND || OSS path ⟹ fflags |= LFLG_APPEND.
- rawmidi_file = kmalloc; user_pversion = 0.
- Loop (waiting for available substream):
  - subdevice = snd_ctl_get_preferred_subdevice(card, SND_CTL_SUBDEV_RAWMIDI) (-1 = any).
  - rawmidi_open_priv(rmidi, subdevice, fflags, rawmidi_file).
  - 0 ⟹ break.
  - -EAGAIN: if O_NONBLOCK ⟹ -EBUSY; else add_wait_queue(rmidi.open_wait); TASK_INTERRUPTIBLE; mutex_unlock; schedule; relock. card.shutdown ⟹ -ENODEV; signal_pending ⟹ -ERESTARTSYS.
  - other err ⟹ break.
- file.private_data = rawmidi_file; mutex_unlock.

REQ-7: rawmidi_open_priv(rmidi, subdevice, mode, rfile):
- For each requested direction (mode & LFLG_INPUT and/or LFLG_OUTPUT):
  - assign_substream(rmidi, subdevice, stream, mode, &sub):
    - info_flags must include INFO_INPUT/INFO_OUTPUT or -ENXIO.
    - subdevice >= 0 ⟹ ensure within substream_count or -ENODEV.
    - Walk substreams: skip opened unless OUTPUT && LFLG_APPEND && substream.append; pick first matching subdevice or any if subdevice < 0; return -EAGAIN if none.
  - open_substream(rmidi, sub, mode):
    - First opener: snd_rawmidi_runtime_create (alloc runtime + PAGE_SIZE buffer kvzalloc, align=3 if UMP); call rmidi.ops.open(sub); on failure free runtime.
    - Under sub.lock: opened=1, active_sensing=0, append=(mode & LFLG_APPEND), pid=get_pid(current); stream.substream_opened++.
    - use_count++.
- rfile = { rmidi, input: sinput, output: soutput }.

REQ-8: Close path: snd_rawmidi_release / rawmidi_release_priv:
- Per-direction close_substream(rmidi, sub, cleanup=1):
  - --use_count > 0 ⟹ return (still other openers).
  - cleanup:
    - INPUT: snd_rawmidi_input_trigger(sub, 0).
    - OUTPUT: if active_sensing: snd_rawmidi_kernel_write(0xFE, 1) (MIDI active-sense shutdown byte). Then drain_output (waits 10s for ring empty); if -ERESTARTSYS ⟹ snd_rawmidi_output_trigger(0).
    - snd_rawmidi_buffer_ref_sync(sub): wait HZ ticks for buffer_ref to drop to 0 (timeout warns "Buffer ref sync timeout").
  - Under sub.lock: opened=0, append=0.
  - sub.ops.close(sub).
  - runtime.private_free?(sub); snd_rawmidi_runtime_free; put_pid; stream.substream_opened--.
- wake_up(rmidi.open_wait) so any process waiting in open path retries.
- kfree(rfile); snd_card_file_remove; module_put.

REQ-9: Ring buffer model (per-direction semantics):
- buffer_size: kvzalloc'd, default PAGE_SIZE, params-settable in [32, 1 MiB], must satisfy `buffer_size & align == 0` (UMP).
- INPUT direction:
  - hw_ptr: where rmidi.receive writes (hardware-irq side).
  - appl_ptr: where userspace read consumes.
  - avail = bytes available to read (initialized to 0 on open).
  - On overrun (avail == buffer_size and more bytes arriving): runtime.xruns += dropped bytes.
- OUTPUT direction:
  - appl_ptr: where userspace write deposits.
  - hw_ptr: where transmit_peek hands to driver / hardware reads.
  - avail = free-space bytes available to write (initialized to buffer_size on open).
- Both ptrs advance modulo buffer_size; lock = spin_lock(sub.lock) protects all four.

REQ-10: snd_rawmidi_input_params(sub, params):
- snd_rawmidi_drain_input(sub) first (trigger off + reset_runtime_ptrs).
- Under sub.rmidi.open_mutex:
  - framing = params.mode & MODE_FRAMING_MASK (7<<0); clock_type = params.mode & MODE_CLOCK_MASK (7<<3).
  - framing == NONE && clock_type != NONE ⟹ -EINVAL.
  - clock_type > MONOTONIC_RAW ⟹ -EINVAL.
  - framing > TSTAMP ⟹ -EINVAL.
  - resize_runtime_buffer(sub, params, is_input=true).
  - sub.framing = framing; sub.clock_type = clock_type.

REQ-11: snd_rawmidi_output_params(sub, params):
- snd_rawmidi_drain_output(sub) first.
- Under open_mutex: if append-mode && use_count > 1 ⟹ -EBUSY (other openers may rely on current buffer).
- resize_runtime_buffer(sub, params, is_input=false).
- sub.active_sensing = !params.no_active_sensing.

REQ-12: resize_runtime_buffer(sub, params, is_input):
- params.buffer_size ∈ [32, 1 MiB]; framing == TSTAMP ⟹ (buffer_size & 0x1f) == 0 (32-byte frame alignment); params.avail_min ∈ [1, buffer_size]; buffer_size & align == 0.
- If buffer_size != current: kvzalloc newbuf. Under sub.lock: buffer_ref > 0 ⟹ -EBUSY (cannot resize while reader/writer in progress); else swap buffer, __reset_runtime_ptrs(is_input). kvfree oldbuf.
- runtime.avail_min = params.avail_min.

REQ-13: snd_rawmidi_ioctl(file, cmd, arg) — cmd-byte must be 'W':
- SNDRV_RAWMIDI_IOCTL_PVERSION (`_IOR('W',0x00,int)`): put_user(SNDRV_RAWMIDI_VERSION = 2.0.5).
- SNDRV_RAWMIDI_IOCTL_INFO (`_IOR('W',0x01)`): per-stream snd_rawmidi_info_user.
- SNDRV_RAWMIDI_IOCTL_USER_PVERSION (`_IOW('W',0x02)`): record userspace protocol version (gates older PARAMS modes).
- SNDRV_RAWMIDI_IOCTL_PARAMS (`_IOWR('W',0x10)`): if user_pversion < 2.0.2 zero params.mode + params.reserved; then output_params or input_params.
- SNDRV_RAWMIDI_IOCTL_STATUS (`_IOWR('W',0x20)`): handled via STATUS32 (32-bit user) or STATUS64 (64-bit user) — switch on size of struct.
- SNDRV_RAWMIDI_IOCTL_DROP (`_IOW('W',0x30,int)`): val ∈ {STREAM_OUTPUT} ⟹ drop_output (discard pending tx).
- SNDRV_RAWMIDI_IOCTL_DRAIN (`_IOW('W',0x31,int)`): val ∈ {STREAM_OUTPUT|STREAM_INPUT} ⟹ drain_output (block 10s) or drain_input (immediate clear).
- Otherwise: rmidi.ops.ioctl if set; else -ENOTTY.

REQ-14: snd_rawmidi_control_ioctl (registered with control core):
- SNDRV_CTL_IOCTL_RAWMIDI_NEXT_DEVICE: walk rmidi list, find next non-UMP device.
- SNDRV_CTL_IOCTL_UMP_NEXT_DEVICE (CONFIG_SND_UMP): walk rmidi list, find next UMP device.
- SNDRV_CTL_IOCTL_UMP_ENDPOINT_INFO / _UMP_BLOCK_INFO: lookup rmidi by info.device; dispatch to rmidi.ops.ioctl with SNDRV_UMP_IOCTL_ENDPOINT_INFO / _BLOCK_INFO.
- SNDRV_CTL_IOCTL_RAWMIDI_PREFER_SUBDEVICE: store preferred subdevice index in control file (consulted by open path).
- SNDRV_CTL_IOCTL_RAWMIDI_INFO: snd_rawmidi_info_select_user.

REQ-15: snd_rawmidi_receive(sub, buffer, count) — called from hardware-irq:
- Under sub.lock.
- !sub.opened ⟹ -EBADFD.
- runtime || runtime.buffer absent ⟹ -EINVAL.
- count = get_aligned_size(runtime, count) (UMP-mask off bottom bits).
- framing == TSTAMP: receive_with_tstamp_framing — for each 32 B frame: check free-space, fill tv_sec/tv_nsec from get_framing_tstamp (REALTIME/MONOTONIC/MONOTONIC_RAW), copy up to 16 B data + length, advance hw_ptr += 32, avail += 32. Xruns += unread tail.
- Else fast 1-byte path: copy 1 byte; xruns++ if avail == buffer_size.
- Else multi-byte: split copy across ring wrap; track count1 ≤ min(buffer_size - hw_ptr, count, buffer_size - avail); aligned via get_aligned_size. Drop excess to xruns.
- If result > 0: runtime.event ⟹ schedule_work(event_work) (deferred kernel callback); else if __snd_rawmidi_ready ⟹ wake_up(runtime.sleep).

REQ-16: snd_rawmidi_transmit_peek(sub, buffer, count) — called from driver-tx-irq:
- Under sub.lock; !sub.opened || !sub.runtime ⟹ -EBADFD.
- runtime.buffer == NULL ⟹ -EINVAL.
- avail >= buffer_size ⟹ return 0 (nothing pending; caller MUST trigger hw down).
- 1-byte path: *buffer = runtime.buffer[hw_ptr]; return 1.
- Multi-byte path: copy split across wrap; get_aligned_size on chunks.
- Returns bytes copied; caller must call snd_rawmidi_transmit_ack(sub, n) after hw actually consumed.

REQ-17: snd_rawmidi_transmit_ack(sub, count):
- Under sub.lock; !sub.opened || !sub.runtime ⟹ -EBADFD.
- snd_BUG_ON(runtime.avail + count > buffer_size).
- count = get_aligned_size(runtime, count).
- hw_ptr += count; hw_ptr %= buffer_size; avail += count; sub.bytes += count.
- If count > 0 && (runtime.drain || __snd_rawmidi_ready): wake_up(runtime.sleep).

REQ-18: snd_rawmidi_transmit(sub, buffer, count):
- Combined peek + ack under one spinlock.

REQ-19: snd_rawmidi_proceed(sub) — driver error-recovery:
- If output ring non-empty: ack the whole pending, return count.
- Used by driver to discard "stuck" data after hw fault.

REQ-20: snd_rawmidi_read(file, buf, count) — userspace input read:
- sub = rfile.input ?: -EIO.
- snd_rawmidi_input_trigger(sub, 1) (start hardware rx).
- Loop while count > 0:
  - Under sub.lock: while !__snd_rawmidi_ready (avail < avail_min):
    - O_NONBLOCK || result > 0 ⟹ -EAGAIN/result.
    - add_wait_queue(runtime.sleep); TASK_INTERRUPTIBLE; spin_unlock; schedule.
    - On wake: card.shutdown ⟹ -ENODEV; signal_pending ⟹ -ERESTARTSYS.
  - snd_rawmidi_kernel_read1(sub, userbuf, NULL, count): per-loop step — bound count1 ≤ min(buffer_size - appl_ptr, count, avail); buffer_ref++; advance appl_ptr; spin_unlock for copy_to_user; spin_lock back; buffer_ref--; -EFAULT on copy fail.
- Return result.

REQ-21: snd_rawmidi_write(file, buf, count) — userspace output write:
- sub = rfile.output; runtime = sub.runtime.
- append && count > buffer_size ⟹ -EIO (atomic-message contract violated).
- Loop while count > 0:
  - Under sub.lock: while !snd_rawmidi_ready_append(sub, count): O_NONBLOCK ⟹ -EAGAIN; else 30 s schedule_timeout. card.shutdown ⟹ -ENODEV; signal_pending ⟹ -ERESTARTSYS. !avail && !timeout ⟹ -EIO.
  - snd_rawmidi_kernel_write1(sub, userbuf, NULL, count): bound count1 ≤ min(buffer_size - appl_ptr, count, avail); buffer_ref++; advance appl_ptr; spin_unlock for copy_from_user; spin_lock back; buffer_ref--.
  - At end-of-loop iteration: if avail < buffer_size ⟹ snd_rawmidi_output_trigger(sub, 1).
- O_DSYNC: drain — while avail != buffer_size: 30 s schedule_timeout; signal/stall ⟹ -ERESTARTSYS/-EIO.

REQ-22: snd_rawmidi_poll:
- INPUT: snd_rawmidi_input_trigger(1); poll_wait(runtime.sleep). EPOLLIN | EPOLLRDNORM if snd_rawmidi_ready(input).
- OUTPUT: poll_wait(runtime.sleep). EPOLLOUT | EPOLLWRNORM if snd_rawmidi_ready(output).

REQ-23: snd_rawmidi_drain_output(sub):
- Under sub.lock: !opened || !runtime || !buffer ⟹ -EINVAL; buffer_ref++; runtime.drain = 1.
- wait_event_interruptible_timeout(runtime.sleep, avail >= buffer_size, 10 * HZ).
- signal_pending ⟹ -ERESTARTSYS; (avail < buffer_size && !timeout) ⟹ -EIO (warn).
- runtime.drain = 0.
- If not interrupted: sub.ops.drain (if defined) else msleep(50); then snd_rawmidi_drop_output.
- buffer_unref.

REQ-24: snd_rawmidi_drain_input(sub):
- input_trigger(0); reset_runtime_ptrs (input mode = avail 0); return 0.

REQ-25: snd_rawmidi_drop_output(sub):
- output_trigger(0); reset_runtime_ptrs (output mode = avail buffer_size); return 0.

REQ-26: snd_rawmidi_info(sub, info):
- info->card = rmidi.card.number; info->device = rmidi.device; info->subdevice = sub.number; info->stream = sub.stream; info->flags = rmidi.info_flags (| INFO_STREAM_INACTIVE if sub.inactive).
- info->id, info->name, info->subname filled via strscpy.
- info->subdevices_count = pstr.substream_count.
- info->subdevices_avail = substream_count - substream_opened.
- info->tied_device = rmidi.tied_device.

REQ-27: snd_rawmidi_status:
- OUTPUT: status.stream = OUTPUT; status.avail = runtime.avail (free-space).
- INPUT: status.stream = INPUT; status.avail = runtime.avail (readable); status.xruns = runtime.xruns; runtime.xruns ← 0 (read-and-clear).
- STATUS32 vs STATUS64: 64-bit kernel marshals tstamp/avail/xruns through the wider struct (REQ-31).

REQ-28: SNDRV_RAWMIDI_LFLG_* (kernel-internal mode flags):
- LFLG_OUTPUT (1<<0), LFLG_INPUT (1<<1), LFLG_OPEN (3<<0), LFLG_APPEND (1<<2).

REQ-29: SNDRV_RAWMIDI_INFO_* (UAPI):
- INFO_OUTPUT (0x1), INFO_INPUT (0x2), INFO_DUPLEX (0x4), INFO_UMP (0x8), INFO_STREAM_INACTIVE (0x10).

REQ-30: SNDRV_RAWMIDI_MODE_* (UAPI input params):
- FRAMING_MASK (7<<0): NONE (0), TSTAMP (1).
- CLOCK_MASK (7<<3): NONE, REALTIME (1<<3), MONOTONIC (2<<3), MONOTONIC_RAW (3<<3).
- FRAMING_DATA_LENGTH = 16 (per frame data bytes).
- struct snd_rawmidi_framing_tstamp = 32 bytes: tv_sec, tv_nsec, length, reserved, data[16]. BUILD_BUG_ON enforces frame_size == 0x20.

REQ-31: SNDRV_RAWMIDI_IOCTL_STATUS (REQ-13) 32/64-bit dual:
- Both share command number `_IOWR('W', 0x20)`; module reads which struct the caller used via a separate compat path.
- snd_rawmidi_status32 { s32 stream; s32 tstamp_sec; s32 tstamp_nsec; u32 avail; u32 xruns; }.
- snd_rawmidi_status64 { int stream; u8 rsvd[4]; s64 tstamp_sec; s64 tstamp_nsec; size_t avail; size_t xruns; }.

REQ-32: UMP (Universal MIDI Packet) — CONFIG_SND_UMP:
- info_flags includes SNDRV_RAWMIDI_INFO_UMP for UMP endpoints.
- Device node `umpC<card>D<device>` (set by snd_rawmidi_init based on rawmidi_is_ump).
- runtime.align = 3 (4-byte alignment); all buffer-size, receive, transmit lengths are masked to ~3 via get_aligned_size.
- Endpoint/Block info via SNDRV_CTL_IOCTL_UMP_ENDPOINT_INFO / _UMP_BLOCK_INFO (dispatched to rmidi.ops.ioctl with SNDRV_UMP_IOCTL_ENDPOINT_INFO / _BLOCK_INFO).
- next-device walk via SNDRV_CTL_IOCTL_UMP_NEXT_DEVICE.

REQ-33: snd_rawmidi_new(card, id, device, output_count, input_count, **rrawmidi):
- kzalloc rmidi; snd_rawmidi_init(rmidi, card, id, device, output_count, input_count, info_flags=0); on failure snd_rawmidi_free.

REQ-34: snd_rawmidi_init (used by both legacy snd_rawmidi_new and UMP path):
- rmidi.card = card; rmidi.device = device.
- mutex_init(open_mutex); init_waitqueue_head(open_wait).
- INIT_LIST_HEAD(streams[INPUT].substreams) / streams[OUTPUT].substreams.
- info_flags set.
- snd_device_alloc(&rmidi.dev).
- dev_set_name: `umpC%iD%i` if UMP else `midiC%iD%i`.
- snd_rawmidi_alloc_substreams for INPUT (input_count) and OUTPUT (output_count).
- snd_device_new(card, SNDRV_DEV_RAWMIDI, rmidi, &ops{dev_free, dev_register, dev_disconnect}).

REQ-35: snd_rawmidi_alloc_substreams(rmidi, stream, direction, count):
- Per-idx: kzalloc snd_rawmidi_substream; substream.stream/number/rmidi/pstr set; spin_lock_init(substream.lock); list_add_tail; stream.substream_count++.

REQ-36: snd_rawmidi_set_ops(rmidi, stream, ops):
- For each substream in rmidi.streams[stream].substreams: substream.ops = ops.

REQ-37: snd_rawmidi_dev_register (snd_device callback):
- rmidi.device < SNDRV_RAWMIDI_DEVICES.
- Under register_mutex: snd_rawmidi_search(card, device) ⟹ -EBUSY; else list_add to global snd_rawmidi_devices.
- snd_register_device(SNDRV_DEVICE_TYPE_RAWMIDI, ...) creates char dev with snd_rawmidi_f_ops.
- rmidi.ops.dev_register optional callback.
- CONFIG_SND_OSSEMUL: if (int)device == midi_map[card.number] or amidi_map: snd_register_oss_device(SNDRV_OSS_DEVICE_TYPE_MIDI) (for /dev/midi* legacy).
- /proc/asound/cardN/midi<dev> via snd_info_create_card_entry + snd_rawmidi_proc_info_read.
- CONFIG_SND_SEQUENCER: if !rmidi.ops.dev_register: snd_seq_device_new(MIDISYNTH) so the ALSA sequencer sees a port.

REQ-38: snd_rawmidi_dev_disconnect:
- Under register_mutex + open_mutex.
- wake_up(open_wait).
- list_del_init.
- For each direction substream with runtime: wake_up(runtime.sleep) (unblock any read/write).
- OSS unregister if registered.
- snd_unregister_device(rmidi.dev).

REQ-39: snd_rawmidi_free:
- snd_info_free_entry(proc_entry).
- rmidi.ops.dev_unregister?(rmidi).
- Free substreams via snd_rawmidi_free_substreams.
- rmidi.private_free?(rmidi).
- put_device(rmidi.dev); kfree(rmidi).

REQ-40: snd_rawmidi_kernel_open / _release — kernel client (sound/core/seq/seq_midi.c):
- kernel_open: try_module_get(card.module) then rawmidi_open_priv under open_mutex; module_put on err.
- kernel_release: rawmidi_release_priv (no card.file_remove, no fd cleanup).

REQ-41: snd_rawmidi_proc_info_read:
- Per-direction: print "Output N" / "Input N", Tx/Rx bytes, owner pid, mode (OSS-compatible / native), buffer_size, avail, framing/clock (input). UMP-only label "Type: UMP" if applicable.

REQ-42: alsa_rawmidi_init / _exit (module init):
- Register snd_rawmidi_control_ioctl on snd_control_ioctls (and compat list) so /dev/snd/controlC<n> answers RAWMIDI_* and UMP_* requests.
- Validate midi_map[] / amidi_map[] indices < SNDRV_RAWMIDI_DEVICES.

## Acceptance Criteria

- [ ] AC-1: snd_rawmidi_new(card, id, device, out_count, in_count) creates rmidi with `midiC<card>D<device>` (or `umpC*D*`) char dev.
- [ ] AC-2: snd_rawmidi_open with FMODE_READ only ⟹ rfile.input populated, rfile.output == NULL.
- [ ] AC-3: snd_rawmidi_open with FMODE_RDWR + O_APPEND on output ⟹ append-mode multi-open allowed (use_count > 1).
- [ ] AC-4: snd_rawmidi_open without O_APPEND on already-opened-output ⟹ -EAGAIN, blocks (or -EBUSY on O_NONBLOCK).
- [ ] AC-5: snd_rawmidi_release with active_sensing on OUTPUT ⟹ writes single 0xFE byte before draining.
- [ ] AC-6: snd_rawmidi_release runs drain_output (10s timeout) before close.
- [ ] AC-7: SNDRV_RAWMIDI_IOCTL_PARAMS: buffer_size 32 ⟹ accepted; buffer_size 16 ⟹ -EINVAL.
- [ ] AC-8: SNDRV_RAWMIDI_IOCTL_PARAMS with framing=TSTAMP and buffer_size not multiple of 32 ⟹ -EINVAL.
- [ ] AC-9: SNDRV_RAWMIDI_IOCTL_PARAMS resize while reader mid-loop (buffer_ref > 0) ⟹ -EBUSY.
- [ ] AC-10: snd_rawmidi_receive over avail==buffer_size ⟹ runtime.xruns increments by overflow.
- [ ] AC-11: snd_rawmidi_receive with framing=TSTAMP writes 32 B framing record per ≤16 B input chunk.
- [ ] AC-12: snd_rawmidi_read blocks for avail >= avail_min; on signal returns -ERESTARTSYS.
- [ ] AC-13: snd_rawmidi_write blocks 30 s for free-space; timeout ⟹ -EIO.
- [ ] AC-14: snd_rawmidi_write with O_DSYNC blocks until ring empties before return.
- [ ] AC-15: SNDRV_RAWMIDI_IOCTL_DROP STREAM_OUTPUT ⟹ ring cleared, hw trigger off.
- [ ] AC-16: SNDRV_RAWMIDI_IOCTL_DRAIN STREAM_OUTPUT blocks up to 10 s for ring to empty.
- [ ] AC-17: snd_rawmidi_poll on input returns EPOLLIN once avail >= avail_min.
- [ ] AC-18: SNDRV_RAWMIDI_IOCTL_INFO populates card / device / subdevice / stream / flags / counts.
- [ ] AC-19: UMP rmidi (info_flags & INFO_UMP) ⟹ runtime.align == 3; all IO masked to 4-byte boundaries.
- [ ] AC-20: SNDRV_CTL_IOCTL_RAWMIDI_NEXT_DEVICE walks legacy midi devices; UMP_NEXT_DEVICE walks UMP devices.

## Architecture

```
struct RawMidi {
  list: ListLink,                            // snd_rawmidi_devices (global, register_mutex)
  card: *mut SndCard,
  device: i32,
  streams: [RawMidiStr; 2],                  // INPUT, OUTPUT
  info_flags: u32,                           // SNDRV_RAWMIDI_INFO_*
  id: [u8; 64],
  name: [u8; 80],
  open_mutex: Mutex<()>,
  open_wait: WaitQueue,
  ops: Option<&'static RawMidiOps>,          // rmidi-level (ioctl, dev_*, proc_read)
  private_data: *mut (),
  private_free: Option<fn(&mut RawMidi)>,
  proc_entry: Option<*mut InfoEntry>,
  dev: *mut Device,
  tied_device: i32,
}

struct RawMidiStr {
  substreams: List<RawMidiSubstream>,
  substream_count: u32,
  substream_opened: u32,
}

struct RawMidiSubstream {
  list: ListLink,
  number: i32,                               // subdevice
  stream: u32,                               // INPUT|OUTPUT
  name: [u8; 32],
  rmidi: *mut RawMidi,
  pstr: *mut RawMidiStr,
  runtime: Option<Box<RawMidiRuntime>>,
  ops: &'static RawMidiSubstreamOps,         // per-direction open/close/trigger/drain
  lock: SpinLock,
  use_count: u32,
  opened: bool,
  append: bool,
  active_sensing: bool,
  framing: u32,                              // MODE_FRAMING_*
  clock_type: u32,                           // MODE_CLOCK_*
  inactive: bool,
  pid: Pid,
  bytes: u64,
}

struct RawMidiRuntime {
  substream: *mut RawMidiSubstream,
  buffer: KvBox<[u8]>,                       // kvzalloc
  buffer_size: usize,                        // default PAGE_SIZE, [32, 1 MiB]
  appl_ptr: usize,                           // userspace
  hw_ptr: usize,                             // kernel/hw
  avail: usize,                              // read-side: readable; write-side: free
  avail_min: usize,
  xruns: usize,                              // read-and-clear on STATUS
  drain: bool,
  buffer_ref: u32,                           // pin count for resize-safety
  align: u8,                                 // 0 (legacy) or 3 (UMP)
  event: Option<fn(&RawMidiSubstream)>,
  event_work: WorkStruct,
  sleep: WaitQueue,
  oss: bool,
  private_data: *mut (),
  private_free: Option<fn(&mut RawMidiSubstream)>,
}

struct RawMidiFile {
  rmidi: *mut RawMidi,
  input: Option<*mut RawMidiSubstream>,
  output: Option<*mut RawMidiSubstream>,
  user_pversion: u32,
}
```

`RawMidi::open(inode, file) -> Result<(), i32>`:
1. (O_APPEND && !O_NONBLOCK) ⟹ -EINVAL.
2. stream_open(inode, file).
3. Look up rmidi by minor (snd_major ⟹ SNDRV_DEVICE_TYPE_RAWMIDI; SOUND_MAJOR ⟹ OSS path under CONFIG_SND_OSSEMUL).
4. try_module_get(rmidi.card.module).
5. mutex_lock(rmidi.open_mutex); snd_card_file_add.
6. fflags = file_flags(file); add LFLG_APPEND on O_APPEND || SOUND_MAJOR.
7. rfile = kmalloc.
8. Loop:
   a. subdevice = snd_ctl_get_preferred_subdevice(card, SND_CTL_SUBDEV_RAWMIDI).
   b. open_priv(rmidi, subdevice, fflags, rfile).
   c. 0 ⟹ break; -EAGAIN && O_NONBLOCK ⟹ -EBUSY; other err ⟹ break.
   d. add_wait_queue(rmidi.open_wait); TASK_INTERRUPTIBLE; mutex_unlock; schedule; relock.
   e. shutdown ⟹ -ENODEV; signal ⟹ -ERESTARTSYS.
9. file.private_data = rfile.

`RawMidi::open_priv(rmidi, subdevice, mode, rfile) -> Result<(), i32>`:
1. rfile.input = rfile.output = None.
2. mode & LFLG_INPUT ⟹ assign_substream(INPUT) → sinput; ? -ENXIO/-ENODEV/-EAGAIN.
3. mode & LFLG_OUTPUT ⟹ assign_substream(OUTPUT) → soutput.
4. sinput ⟹ open_substream(INPUT).
5. soutput ⟹ open_substream(OUTPUT); on err and sinput ⟹ close_substream(input, cleanup=0) (no drain/wait_sync since never used).
6. rfile.rmidi = rmidi; rfile.input = sinput; rfile.output = soutput.

`RawMidi::receive(sub, buffer, count) -> i32`:
1. spin_lock_irqsave(sub.lock).
2. !sub.opened ⟹ -EBADFD.
3. runtime missing ⟹ -EINVAL.
4. count = get_aligned_size(runtime, count).
5. count == 0 ⟹ return 0.
6. sub.framing == TSTAMP ⟹ receive_with_tstamp_framing (32 B per ≤16 B chunk, frame.tv_sec/tv_nsec from get_framing_tstamp).
7. Else fast-path (count==1) ⟹ store byte at hw_ptr or xruns++.
8. Else multi-byte ⟹ split write across ring wrap; xruns += unwritten.
9. If result > 0:
   a. runtime.event ⟹ schedule_work(runtime.event_work).
   b. Else __snd_rawmidi_ready ⟹ wake_up(runtime.sleep).
10. spin_unlock_irqrestore.

`RawMidi::read(file, buf, count) -> ssize_t`:
1. sub = rfile.input ?: -EIO.
2. input_trigger(sub, 1).
3. result = 0.
4. While count > 0:
   a. spin_lock_irq(sub.lock).
   b. While !__snd_rawmidi_ready:
      - O_NONBLOCK || result > 0 ⟹ -EAGAIN/result.
      - add_wait_queue(runtime.sleep); TASK_INTERRUPTIBLE; spin_unlock; schedule.
      - card.shutdown ⟹ -ENODEV; signal ⟹ -ERESTARTSYS.
      - spin_lock_irq.
      - If !avail post-wake: -EIO.
   c. spin_unlock_irq.
   d. count1 = kernel_read1(sub, userbuf=buf, kernelbuf=NULL, count) — see REQ-20.
   e. result += count1; buf += count1; count -= count1.
5. Return result.

`RawMidi::write(file, buf, count) -> ssize_t`:
1. sub = rfile.output; runtime = sub.runtime.
2. append && count > buffer_size ⟹ -EIO.
3. result = 0.
4. While count > 0:
   a. spin_lock_irq(sub.lock).
   b. While !snd_rawmidi_ready_append(sub, count):
      - O_NONBLOCK ⟹ -EAGAIN/result.
      - 30 s schedule_timeout; signal/shutdown handled.
      - !avail && !timeout ⟹ -EIO.
   c. spin_unlock_irq.
   d. count1 = kernel_write1(sub, userbuf=buf, kernelbuf=NULL, count).
   e. result += count1; buf += count1; count -= count1.
   f. (count1 < count && O_NONBLOCK) ⟹ break.
5. O_DSYNC: spin-loop with 30 s schedule_timeout until avail == buffer_size; signal/stall handled.

`RawMidi::drain_output(sub) -> Result<(), i32>`:
1. spin_lock_irq(sub.lock); !opened || !runtime || !buffer ⟹ -EINVAL.
2. buffer_ref++; runtime.drain = 1; spin_unlock_irq.
3. wait_event_interruptible_timeout(runtime.sleep, avail >= buffer_size, 10 * HZ).
4. spin_lock_irq:
   a. signal_pending ⟹ err = -ERESTARTSYS.
   b. (avail < buffer_size && !timeout) ⟹ err = -EIO (rmidi_warn).
   c. runtime.drain = 0.
5. spin_unlock_irq.
6. err != -ERESTARTSYS ⟹ sub.ops.drain?(sub) ?: msleep(50); snd_rawmidi_drop_output(sub).
7. spin_lock_irq; buffer_ref--; spin_unlock_irq.

`RawMidi::transmit(sub, buffer, count) -> i32`:
1. spin_lock_irqsave(sub.lock).
2. !opened ⟹ -EBADFD.
3. n = __snd_rawmidi_transmit_peek(sub, buffer, count).
4. n <= 0 ⟹ return n.
5. return __snd_rawmidi_transmit_ack(sub, n).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `runtime_buffer_size_bounded` | INVARIANT | per-resize_runtime_buffer: buffer_size ∈ [32, 1 MiB] and aligned to `align`. |
| `framing_tstamp_buffer_aligned` | INVARIANT | per-resize_runtime_buffer with framing == TSTAMP: buffer_size % 32 == 0. |
| `avail_in_range` | INVARIANT | per-substream under lock: avail ∈ [0, buffer_size]. |
| `hw_ptr_modulo_buffer_size` | INVARIANT | per-receive / transmit_ack: hw_ptr < buffer_size. |
| `appl_ptr_modulo_buffer_size` | INVARIANT | per-read / write: appl_ptr < buffer_size. |
| `buffer_ref_balanced` | INVARIANT | per-kernel_read1/write1: buffer_ref incremented before copy and decremented in any exit path. |
| `resize_requires_buffer_ref_zero` | INVARIANT | per-resize_runtime_buffer: swap only if buffer_ref == 0; else -EBUSY. |
| `xruns_monotonic_until_status_read` | INVARIANT | per-status_input: xruns ← 0 after copy_to_user; receive's xruns += only adds. |
| `use_count_balanced` | INVARIANT | per-open_substream / close_substream: use_count delta = +1/-1 mirrored. |
| `frame_size_const_32` | INVARIANT | BUILD_BUG_ON(sizeof(snd_rawmidi_framing_tstamp) != 32). |
| `ump_align_3` | INVARIANT | per-rawmidi_is_ump rmidi at runtime_create: runtime.align == 3. |

### Layer 2: TLA+

`sound/core/rawmidi.tla`:
- Per-substream model = (closed | opened) × (input ring state) × (output ring state) × (set of open fds).
- Actions: open, close, read, write, receive, transmit_peek_ack, params_resize, drop, drain, disconnect.
- Properties:
  - `safety_single_owner_non_append_output` — per-OUTPUT substream: !append ⟹ |opened fds| ≤ 1.
  - `safety_resize_blocked_during_io` — per-resize_runtime_buffer: buffer_ref > 0 at start ⟹ -EBUSY (no concurrent buffer swap with in-flight copy).
  - `safety_avail_consistent_with_ring` — per-substream: avail == (hw_ptr - appl_ptr) mod buffer_size for INPUT; == buffer_size - that for OUTPUT.
  - `safety_xruns_only_on_full` — per-receive: xruns += k only if write would have overflowed (avail == buffer_size for those k bytes).
  - `safety_drain_completes_or_times_out` — per-drain_output: returns in ≤ 10 s with avail == buffer_size or -EIO/-ERESTARTSYS.
  - `liveness_read_eventually_unblocks` — per-blocked read: either receive ⟹ wake, disconnect ⟹ -ENODEV, or signal ⟹ -ERESTARTSYS within finite steps.
  - `liveness_disconnect_wakes_all_waiters` — per-dev_disconnect: every waiter on runtime.sleep and open_wait is woken.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `RawMidi::new` post: streams[0..2] non-empty; dev_name == "midiC%dD%d" or "umpC%dD%d" | `RawMidi::new` / `init` |
| `RawMidi::open` post: success ⟹ rfile.input or rfile.output non-NULL; module_get balanced | `RawMidi::open` |
| `RawMidi::release` post: sub.use_count == 0 ⟹ runtime freed; ops.close called; pid put | `RawMidi::release` |
| `RawMidi::receive` post: result == bytes written to ring; xruns += dropped | `RawMidi::receive` |
| `RawMidi::transmit_peek` post: returns ≤ min(count, used-ring-bytes); does not advance hw_ptr | `RawMidi::transmit_peek` |
| `RawMidi::transmit_ack` post: hw_ptr += count (mod size); avail += count; wake on drain or ready | `RawMidi::transmit_ack` |
| `RawMidi::read` post: copy_to_user count, advance appl_ptr; avail -= count | `RawMidi::read` |
| `RawMidi::write` post: copy_from_user count, advance appl_ptr; avail -= count; output_trigger(1) | `RawMidi::write` |
| `RawMidi::drain_output` post: returns within 10 s with avail == buffer_size or -EIO/-ERESTARTSYS | `RawMidi::drain_output` |
| `RawMidi::output_params` post: buffer_size valid; active_sensing set; -EBUSY if append+use_count>1 | `RawMidi::output_params` |
| `RawMidi::input_params` post: framing+clock_type valid; -EINVAL on illegal pairs | `RawMidi::input_params` |

### Layer 4: Verus/Creusot functional

`Per-`alsa-utils amidi`-style userspace: open(/dev/snd/midiC0D0, RDWR) → ioctl(INFO) → ioctl(PARAMS, buffer_size=4096, avail_min=1) → read(N bytes) loop → write(MIDI bytes) → ioctl(DRAIN, OUTPUT) → close` — semantic equivalence with Linux per-Documentation/sound/designs/midi-2.0.rst (UMP) and per-Documentation/sound/cards/serial-u16550.rst (legacy MIDI 1.0).

`Per-`aplaymidi` write path: open(write-only) → ioctl(PARAMS) → write loop with hw triggered on first non-empty → ALSA hw driver calls snd_rawmidi_transmit_peek/_ack from tx-irq → close runs drain` — per-MIDI 1.0 device behavior on USB-MIDI / UART16550 / OPL3 / Roland Sound Canvas drivers.

## Hardening

(Inherits row-1 features from `sound/00-overview.md` § Hardening.)

Raw-MIDI reinforcement:

- **Per-substream spinlock_irq protects ring ptrs** — defense against per-hardware-irq race with userspace read/write.
- **Per-runtime.buffer_ref pin count blocks concurrent resize** — defense against per-resize-while-copy UAF.
- **Per-resize buffer_size ∈ [32, 1 MiB], framed must be 32-aligned** — defense against per-userspace-DoS huge alloc and per-misaligned framing.
- **Per-30 s schedule_timeout on blocking write + 10 s on drain** — defense against per-stuck-hw indefinite hang.
- **Per-snd_rawmidi_buffer_ref_sync HZ-tick loop on close** — defense against per-close-during-irq race; warns "Buffer ref sync timeout" if exceeded.
- **Per-active_sensing 0xFE byte sent on close** — defense against per-stuck-note on hw MIDI hardware.
- **Per-runtime.xruns increments only when avail saturated (drop count to runtime.xruns); read-and-clear on STATUS** — defense against per-silent-data-loss.
- **Per-append-mode requires LFLG_APPEND on every opener** — defense against per-message-interleave from non-append fd.
- **Per-non-append OUTPUT: assign_substream rejects already-opened substream (-EAGAIN)** — defense against per-double-open OUTPUT.
- **Per-UMP align = 3 (4-byte boundary) enforced via get_aligned_size** — defense against per-malformed UMP packet (32-bit packet integrity).
- **Per-snd_rawmidi_dev_disconnect wakes all open_wait + runtime.sleep waiters** — defense against per-stuck-process on device unplug.
- **Per-card.shutdown checked in every blocking loop** — defense against per-card-removal indefinite block.
- **Per-snd_rawmidi_kernel_open module_get balanced with kernel_release module_put** — defense against per-seq_midi-leak.
- **Per-snd_rawmidi_search under register_mutex** — defense against per-concurrent-register conflict.
- **Per-snd_rawmidi_alloc_substreams limits subdevice index < SNDRV_RAWMIDI_DEVICES** — defense against per-out-of-range subdevice.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- sound/core/seq/seq_midi.c sequencer-side MIDI client (covered separately if expanded — uses snd_rawmidi_kernel_open API)
- sound/core/ump.c UMP endpoint helpers (covered separately if expanded)
- sound/core/rawmidi_compat.c 32-bit compat marshalling (FFI-thin)
- sound/usb/midi.c, sound/usb/midi2.c USB-MIDI 1.0 / 2.0 host drivers (per-driver)
- sound/drivers/serial-u16550.c UART16550 MIDI driver (per-driver)
- sound/core/control.c control-fd ioctl multiplexer (covered in `control.md` Tier-3)
- sound/core/pcm.c audio PCM streams (covered in `pcm.md` Tier-3)
- sound/core/oss/ OSS emulation (covered separately if expanded)
- Implementation code
