# Tier-3: sound/core/control.c — ALSA mixer / control API

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: sound/00-overview.md
upstream-paths:
  - sound/core/control.c (~2532 lines)
  - sound/core/control_compat.c (32-bit compat ioctls)
  - include/sound/control.h (struct snd_kcontrol, snd_kcontrol_new, snd_kcontrol_volatile, snd_ctl_layer_ops)
  - include/uapi/sound/asound.h (struct snd_ctl_elem_id, snd_ctl_elem_info, snd_ctl_elem_value, snd_ctl_event, snd_ctl_tlv, SNDRV_CTL_IOCTL_*, SNDRV_CTL_ELEM_*, SNDRV_CTL_EVENT_*)
  - include/sound/tlv.h (Type-Length-Value DB-scale containers)
-->

## Summary

ALSA's **control API** is the kernel-userspace contract for mixer-style audio parameters: volume sliders, channel-routing switches, S/PDIF (IEC958) configuration, enumerated selectors, IEC958 default/professional bits, codec capability registers, and any other "knob" a card driver wants to expose. Per `/dev/snd/controlC<n>` per-card character device. Per-control is a `struct snd_kcontrol` that bundles a `struct snd_ctl_elem_id` (iface + device + subdevice + name + index + numid) with `info()` / `get()` / `put()` callbacks and optional TLV (Type-Length-Value) container. Per-iface dimensions: `SNDRV_CTL_ELEM_IFACE_CARD / HWDEP / MIXER / PCM / RAWMIDI / TIMER / SEQUENCER`. Per-type: `BOOLEAN / INTEGER / ENUMERATED / BYTES / IEC958 / INTEGER64`. Per-event subscription: userspace opens `/dev/snd/controlC<n>`, ioctl(`SNDRV_CTL_IOCTL_SUBSCRIBE_EVENTS`), then `read()` returns `struct snd_ctl_event` records as values change. Per-TLV: out-of-band dB-scale metadata or driver commands. Critical for: alsactl-state preservation across reboot, PulseAudio / PipeWire stream routing, codec driver knob exposition, audio LED layer.

This Tier-3 covers `sound/core/control.c` (~2532 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct snd_kcontrol` | per-card-control instance (id + count + callbacks + vd[]) | `KControl` |
| `struct snd_kcontrol_new` | template for per-driver constant control declaration | `KControlNew` |
| `struct snd_kcontrol_volatile` | per-element volatile state (access flags + owner) | `KControlVolatile` |
| `struct snd_ctl_elem_id` | per-element identifier (iface,device,subdevice,name,index,numid) | `CtlElemId` |
| `struct snd_ctl_elem_info` | per-element type/min/max/step/items metadata | `CtlElemInfo` |
| `struct snd_ctl_elem_value` | per-element current value payload | `CtlElemValue` |
| `struct snd_ctl_event` | per-userspace-read event record | `CtlEvent` |
| `struct snd_ctl_tlv` | per-TLV-ioctl header (numid + length + tlv[]) | `CtlTlv` |
| `struct snd_ctl_file` | per-open-fd state (events, subscription, owned vd) | `CtlFile` |
| `struct snd_kctl_event` | per-kernel-queued event awaiting read | `KctlEvent` |
| `struct snd_ctl_layer_ops` | per-layer (audio-LED etc.) tap interface | `CtlLayerOps` |
| `snd_ctl_new1()` | per-allocate from template | `Ctl::new1` |
| `snd_ctl_free_one()` | per-free orphan (pre-add) | `Ctl::free_one` |
| `snd_ctl_add()` | per-attach exclusive | `Ctl::add` |
| `snd_ctl_replace()` | per-attach replacing | `Ctl::replace` |
| `snd_ctl_remove()` | per-detach + free | `Ctl::remove` |
| `snd_ctl_remove_id()` | per-id-lookup remove | `Ctl::remove_id` |
| `snd_ctl_find_id()` / `snd_ctl_find_numid()` | per-lookup (hash + list fallback) | `Ctl::find_id` / `find_numid` |
| `snd_ctl_rename()` / `snd_ctl_rename_id()` | per-rename in place | `Ctl::rename` / `rename_id` |
| `snd_ctl_activate_id()` | per-INACTIVE flag toggle | `Ctl::activate_id` |
| `snd_ctl_notify()` / `snd_ctl_notify_one()` | per-event queue + wake + LED hook | `Ctl::notify` / `notify_one` |
| `snd_ctl_open()` / `snd_ctl_release()` | per-fd lifecycle | `Ctl::open` / `release` |
| `snd_ctl_ioctl()` | per-fd ioctl multiplexer | `Ctl::ioctl` |
| `snd_ctl_read()` | per-fd event-queue blocking read | `Ctl::read` |
| `snd_ctl_poll()` | per-fd EPOLLIN poll on events | `Ctl::poll` |
| `snd_ctl_elem_list()` | per-card enumeration to userspace | `Ctl::elem_list` |
| `snd_ctl_elem_info()` | per-id info-callback dispatch | `Ctl::elem_info` |
| `snd_ctl_elem_read()` | per-id get-callback dispatch + validation | `Ctl::elem_read` |
| `snd_ctl_elem_write()` | per-id put-callback dispatch + validation + notify | `Ctl::elem_write` |
| `snd_ctl_elem_lock()` / `_unlock()` | per-vd-owner exclusive write lock | `Ctl::elem_lock` / `elem_unlock` |
| `snd_ctl_elem_add()` | per-userspace-defined element (SNDRV_CTL_ELEM_ACCESS_USER) | `Ctl::elem_add` |
| `snd_ctl_elem_remove()` (user) | per-userspace-defined element removal | `Ctl::elem_remove` |
| `snd_ctl_subscribe_events()` | per-fd subscription toggle | `Ctl::subscribe_events` |
| `snd_ctl_tlv_ioctl()` / `call_tlv_handler()` / `read_tlv_buf()` | per-TLV read/write/cmd dispatch | `Ctl::tlv_ioctl` |
| `snd_ctl_register_ioctl()` / `_compat()` / `_unregister*()` | per-subdriver ioctl extension | `Ctl::register_ioctl` |
| `snd_ctl_register_layer()` / `_disconnect_layer()` / `_request_layer()` | per-tap layer (audio LED) | `Ctl::register_layer` |
| `snd_ctl_dev_register()` / `_disconnect()` / `_free()` | per-snd_device callbacks | `Ctl::dev_register` |
| `snd_ctl_create()` | per-card init | `Ctl::create` |
| `snd_ctl_boolean_mono_info()` / `_stereo_info()` / `snd_ctl_enum_info()` | per-driver convenience helpers | `Ctl::boolean_mono_info` etc. |
| `snd_ctl_get_preferred_subdevice()` | per-fd subdevice preference (PCM/RAWMIDI/etc.) | `Ctl::get_preferred_subdevice` |

## Compatibility contract

REQ-1: struct snd_kcontrol fields:
- list: per-card linkage (card.controls).
- id: snd_ctl_elem_id { iface, device, subdevice, name[44], index, numid }.
- count: per-element-range size (id.index..id.index+count is one kctl).
- info: snd_kcontrol_info_t callback (fills snd_ctl_elem_info).
- get: snd_kcontrol_get_t callback (reads snd_ctl_elem_value).
- put: snd_kcontrol_put_t callback (writes snd_ctl_elem_value, returns >0 if changed).
- tlv.p: const u32 * static TLV (SNDRV_CTL_TLVT_*) — used when no callback.
- tlv.c: snd_kcontrol_tlv_rw_t callback (TLV_CALLBACK access).
- private_value: per-driver opaque.
- private_data: per-driver opaque.
- private_free: per-kctl destructor.
- vd[count] flex array of snd_kcontrol_volatile { access, owner } — `__counted_by(count)`.

REQ-2: struct snd_kcontrol_new (driver-static template):
- iface, device, subdevice, name, index, access, count, info/get/put/tlv, private_value.
- access == 0 => SNDRV_CTL_ELEM_ACCESS_READWRITE.
- count == 0 => 1.
- snd_ctl_new1(ncontrol, private_data) allocates kctl, copies fields, masks `access` against the legal subset (READWRITE | VOLATILE | INACTIVE | TLV_READWRITE | TLV_COMMAND | TLV_CALLBACK | LED_MASK | SKIP_CHECK).

REQ-3: SNDRV_CTL_ELEM_IFACE_* (UAPI):
- 0 = CARD (global), 1 = HWDEP, 2 = MIXER, 3 = PCM, 4 = RAWMIDI, 5 = TIMER, 6 = SEQUENCER.
- Per-iface name-space is independent — two elements may share name+index across different ifaces.

REQ-4: SNDRV_CTL_ELEM_TYPE_*:
- BOOLEAN (1): long value 0/1, max 128 values per element.
- INTEGER (2): long value, min/max/step from info, max 128.
- ENUMERATED (3): unsigned int item, items > 0 required, max 128.
- BYTES (4): unsigned char value[512] payload, max 512.
- IEC958 (5): struct snd_aes_iec958 payload, count must be 1.
- INTEGER64 (6): long long value, min/max/step, max 64.

REQ-5: snd_ctl_elem_id { iface, device, subdevice, name[44], index, numid }:
- numid 0 in user request: lookup by (iface, device, subdevice, name, index).
- numid != 0: direct lookup; xa_load(card.ctl_numids).
- numid is auto-assigned at snd_ctl_add() and unique per (card, lifetime).
- For controls with count > 1: numid range = [base, base + count - 1].

REQ-6: snd_ctl_new1(ncontrol, private_data):
- snd_BUG_ON if !ncontrol || !ncontrol.info.
- count = ncontrol.count ?: 1; reject count == 0 || count > MAX_CONTROL_COUNT.
- access = ncontrol.access ?: SNDRV_CTL_ELEM_ACCESS_READWRITE.
- access &= ALLOWED_MASK (see REQ-2).
- kzalloc kctl with flex vd[count].
- Per-idx: vd[idx].access = access; vd[idx].owner = NULL.
- Copy id { iface, device, subdevice, name (strscpy, warn on truncation), index }.
- Copy info/get/put/tlv.p/private_value/private_data.
- numid is set later by snd_ctl_add().

REQ-7: snd_ctl_add / snd_ctl_replace / __snd_ctl_add_replace:
- Take card.controls_rwsem write.
- Reject id.index > UINT_MAX - count (overflow).
- old = snd_ctl_find_id(card, &id).
- mode == CTL_ADD_EXCLUSIVE && old: -EBUSY (log "is already present").
- mode == CTL_REPLACE && !old: -EINVAL.
- old != NULL && (CTL_REPLACE | CTL_ADD_ON_REPLACE): snd_ctl_remove_locked(card, old).
- snd_ctl_find_hole(card, count): advance card.last_numid past any conflicting numid ranges (max 100000 iterations, else -ENOMEM).
- kctl.id.numid = card.last_numid + 1; card.last_numid += count.
- list_add_tail(&kctl.list, &card.controls); card.controls_count += count.
- add_hash_entries(card, kctl): xa_store_range(card.ctl_numids, ...) + per-index xa_insert(card.ctl_hash, get_ctl_id_hash(id)). Hash collision sets card.ctl_hash_collision.
- Per-index: snd_ctl_notify_one(card, SNDRV_CTL_EVENT_MASK_ADD, kctl, idx).
- On failure: snd_ctl_free_one(kctl) (auto-cleanup contract).

REQ-8: snd_ctl_remove / __snd_ctl_remove:
- Take card.controls_rwsem write.
- remove_hash_entries (xa_erase numid range + xa_erase per-name hash if matched).
- Under controls_rwlock write: list_del; card.controls_count -= kctl.count.
- Per-index: snd_ctl_notify_one(card, SNDRV_CTL_EVENT_MASK_REMOVE = ~0U, kctl, idx).
- snd_ctl_free_one(kctl): kctl.private_free?(kctl); kfree(kctl).

REQ-9: snd_ctl_find_id / snd_ctl_find_numid:
- numid != 0: snd_ctl_find_numid(card, numid) — xa_load(card.ctl_numids, numid) under CONFIG_SND_CTL_FAST_LOOKUP; else linear list under read_lock_irqsave(controls_rwlock).
- numid == 0: hash lookup via get_ctl_id_hash(id) (37-multiplier polynomial on iface/device/subdevice/name/index, masked to LONG_MAX). If hash hit and elem_id_matches: return. If hash miss and !card.ctl_hash_collision: return NULL. Else fall through to linear list.
- elem_id_matches: iface==id.iface && device==id.device && subdevice==id.subdevice && !strncmp(name) && kctl.id.index <= id.index < kctl.id.index + kctl.count.

REQ-10: snd_ctl_open(inode, file):
- stream_open(inode, file).
- card = snd_lookup_minor_data(iminor, SNDRV_DEVICE_TYPE_CONTROL) ?: -ENODEV.
- snd_card_file_add(card, file).
- try_module_get(card.module) else -EFAULT.
- ctl = kzalloc(*ctl).
- INIT_LIST_HEAD(events); init_waitqueue_head(change_sleep); spin_lock_init(read_lock).
- ctl.card = card; ctl.pid = get_pid(current); per-i: ctl.preferred_subdevice[i] = -1.
- file.private_data = ctl.
- Under write_lock_irqsave(controls_rwlock): list_add_tail(&ctl.list, &card.ctl_files).

REQ-11: snd_ctl_release(inode, file):
- Detach ctl from card.ctl_files under controls_rwlock.
- Under controls_rwsem write: for each kctl, per-idx clear vd[idx].owner if it equals this ctl (release any held SNDRV_CTL_ELEM_ACCESS_LOCK).
- snd_fasync_free(ctl.fasync); empty event queue; put_pid; kfree(ctl); module_put.

REQ-12: snd_ctl_ioctl(file, cmd, arg) — per-cmd dispatch:
- SNDRV_CTL_IOCTL_PVERSION (`_IOR('U',0x00,int)`): put_user(SNDRV_CTL_VERSION).
- SNDRV_CTL_IOCTL_CARD_INFO (`'U',0x01`): copy_to_user struct snd_ctl_card_info.
- SNDRV_CTL_IOCTL_ELEM_LIST (`'U',0x10`): enumerate IDs.
- SNDRV_CTL_IOCTL_ELEM_INFO (`'U',0x11`): kctl.info callback.
- SNDRV_CTL_IOCTL_ELEM_READ (`'U',0x12`): kctl.get callback under power-ref.
- SNDRV_CTL_IOCTL_ELEM_WRITE (`'U',0x13`): kctl.put callback under power-ref.
- SNDRV_CTL_IOCTL_ELEM_LOCK / _UNLOCK (`'U',0x14`/`0x15`): vd.owner = file or NULL.
- SNDRV_CTL_IOCTL_SUBSCRIBE_EVENTS (`'U',0x16`): toggle ctl.subscribed.
- SNDRV_CTL_IOCTL_ELEM_ADD (`'U',0x17`): user-defined element.
- SNDRV_CTL_IOCTL_ELEM_REPLACE (`'U',0x18`): user-defined element (replace).
- SNDRV_CTL_IOCTL_ELEM_REMOVE (`'U',0x19`): user-defined removal.
- SNDRV_CTL_IOCTL_TLV_READ / _WRITE / _COMMAND (`'U',0x1a`/`0x1b`/`0x1c`): TLV dispatch.
- SNDRV_CTL_IOCTL_POWER (`'U',0xd0`): -ENOPROTOOPT.
- SNDRV_CTL_IOCTL_POWER_STATE (`'U',0xd1`): put_user(SNDRV_CTL_POWER_D0).
- Unknown: walk snd_control_ioctls list (-ENOIOCTLCMD chains to next); finally -ENOTTY.

REQ-13: snd_ctl_elem_info(ctl, info):
- Take controls_rwsem read.
- kctl = snd_ctl_find_id ?: -ENOENT.
- result = kctl.info(kctl, info) (driver populates type/count/min/max/step/items/name[]).
- snd_BUG_ON(info.access) — info callback MUST NOT set access (kernel owns it).
- index_offset = id.index - kctl.id.index; vd = kctl.vd[index_offset].
- info.access = vd.access. If vd.owner: info.access |= LOCK; if owner == ctl: |= OWNER; info.owner = pid_vnr(owner.pid). Else info.owner = -1.
- snd_ctl_check_elem_info validation (unless SKIP_CHECK): type in [BOOLEAN..INTEGER64]; ENUMERATED.items > 0; count <= max_value_counts[type].

REQ-14: snd_ctl_elem_read(card, control):
- Take controls_rwsem read.
- kctl = snd_ctl_find_id ?: -ENOENT.
- vd = kctl.vd[ioff].
- Require vd.access & ACCESS_READ && kctl.get ?: -EPERM.
- Fill remaining bytes of value with pattern 0xdeadbeef (CONFIG_SND_CTL_DEBUG).
- kctl.get(kctl, control).
- Sanity check: int values within [min,max] aligned to step; remaining area unchanged (= pattern) — else -EINVAL "access overflow".

REQ-15: snd_ctl_elem_write(card, file, control):
- Take controls_rwsem write.
- kctl = snd_ctl_find_id ?: -ENOENT.
- vd = kctl.vd[ioff].
- Require vd.access & ACCESS_WRITE && kctl.put && (!vd.owner || vd.owner == file) else -EPERM.
- CONFIG_SND_CTL_INPUT_VALIDATION: __snd_ctl_elem_info + sanity_check_input_values.
- snd_ctl_put dispatches (CONFIG_SND_CTL_DEBUG: put_verify wraps get-then-put-then-compare for trace_snd_ctl_put; else direct kctl.put).
- put return >0 means value changed:
  - downgrade_write to read.
  - snd_ctl_notify_one(card, SNDRV_CTL_EVENT_MASK_VALUE, kctl, ioff).
  - up_read.
- put return 0: up_write (no notify).
- put return <0: up_write; -err.

REQ-16: SNDRV_CTL_ELEM_ACCESS_* flags:
- READ (1<<0), WRITE (1<<1), READWRITE = READ|WRITE.
- VOLATILE (1<<2): may change without notification.
- TLV_READ (1<<4), TLV_WRITE (1<<5), TLV_READWRITE, TLV_COMMAND (1<<6).
- INACTIVE (1<<8): no-op but updates allowed.
- LOCK (1<<9): vd.owner set.
- OWNER (1<<10): vd.owner == this file.
- TLV_CALLBACK (1<<28): kernel-side TLV callback.
- USER (1<<29): userspace-defined element.
- LED_MASK / SKIP_CHECK: layer-specific.

REQ-17: TLV (Type-Length-Value):
- u32 type + u32 length + length bytes payload.
- Standard types in include/sound/tlv.h: SNDRV_CTL_TLVT_DB_SCALE (mB-per-step), DB_LINEAR (min/max mB), DB_RANGE, DB_MINMAX, CONTAINER, etc.
- TLV_READ ioctl: copy kctl.tlv.p[0..2 + tlv.p[1]] to userspace, OR via kctl.tlv.c callback if TLV_CALLBACK.
- TLV_WRITE: kctl.tlv.c(WRITE, ...) only; static tlv.p is read-only.
- TLV_COMMAND: kctl.tlv.c(CMD, ...).
- Per-vd.owner: write/cmd denied unless owner == file (read always allowed if TLV_READ access).
- snd_ctl_tlv_ioctl: copy_from_user header (numid, length); length >= 2*sizeof(u32) required; lookup kctl by numid; vd has TLV_CALLBACK ⟹ call_tlv_handler; else TLV_READ ⟹ read_tlv_buf; else -ENXIO.

REQ-18: snd_ctl_subscribe_events(file, ptr):
- copy_from_user(subscribe).
- subscribe < 0: copy_to_user(file.subscribed) and return.
- subscribe != 0: file.subscribed = 1.
- subscribe == 0 and was subscribed: snd_ctl_empty_read_queue + file.subscribed = 0.

REQ-19: snd_ctl_notify(card, mask, id) (atomic-safe):
- If card.shutdown: return.
- read_lock_irqsave(controls_rwlock).
- For each ctl in card.ctl_files: if !ctl.subscribed: skip.
- Under ctl.read_lock spinlock: coalesce — if existing event with same numid: ev.mask |= mask. Else kzalloc(GFP_ATOMIC) snd_kctl_event { id, mask, list }; list_add_tail to ctl.events. If alloc fails: dev_err "No memory available to allocate event".
- wake_up(ctl.change_sleep); snd_kill_fasync(ctl.fasync, SIGIO, POLL_IN).

REQ-20: snd_ctl_notify_one(card, mask, kctl, ioff):
- id = kctl.id; id.index += ioff; id.numid += ioff.
- snd_ctl_notify(card, mask, &id).
- Under snd_ctl_layer_rwsem read: for each lops in snd_ctl_layer: lops.lnotify(card, mask, kctl, ioff) — audio LED hook.

REQ-21: snd_ctl_read(file, buffer, count, offset):
- Require ctl.subscribed else -EBADFD.
- Require count >= sizeof(struct snd_ctl_event) else -EINVAL.
- Loop while count >= event_size:
  - While events empty: O_NONBLOCK ⟹ -EAGAIN (or partial result); else add_wait_queue(ctl.change_sleep) + schedule. card.shutdown ⟹ -ENODEV; signal_pending ⟹ -ERESTARTSYS.
  - Pop one snd_kctl_event; copy_to_user as snd_ctl_event { type = SNDRV_CTL_EVENT_ELEM, data.elem = { mask, id } }.
  - count -= event_size; result += event_size.
- Return result > 0 ? result : err.

REQ-22: snd_ctl_poll(file, wait):
- ctl.subscribed ?: 0.
- poll_wait(file, &ctl.change_sleep, wait).
- !list_empty(events) ⟹ EPOLLIN | EPOLLRDNORM.

REQ-23: SNDRV_CTL_EVENT_* masks:
- MASK_VALUE (1<<0): value changed (most common).
- MASK_INFO (1<<1): info changed (range/items/access).
- MASK_ADD (1<<2): element added.
- MASK_TLV (1<<3): TLV-tree changed.
- MASK_REMOVE (~0U): element removed.
- event.type = SNDRV_CTL_EVENT_ELEM (0); event.data.elem.mask + id.

REQ-24: snd_ctl_elem_add (user-defined element via ioctl ELEM_ADD/ELEM_REPLACE):
- info.id.name non-empty and not over-long.
- replace: snd_ctl_remove_user_ctl(file, &id).
- count = info.owner ?: 1; reject > MAX_CONTROL_COUNT.
- access masked to (READWRITE | INACTIVE | TLV_WRITE); access |= USER; TLV_WRITE ⟹ |= TLV_CALLBACK.
- snd_ctl_check_elem_info validation; reject count < 1.
- private_size = value_sizes[info.type] * info.count.
- alloc_size = sizeof(struct user_element) + private_size * count.
- check_user_elem_overflow against card.user_ctl_alloc_size + max_user_ctl_alloc_size.
- Take controls_rwsem write.
- snd_ctl_new(&kctl, count, access, file) — file becomes initial vd.owner of each element ("locked from creation").
- kctl.private_data = ue (user_element with elem_data, elem_data_size, tlv_data, priv_data for enum names).
- kctl.private_free = snd_ctl_elem_user_free (decrements card.user_ctl_alloc_size).
- ue.card = card; ue.info = *info; ue.info.access = 0; ue.elem_data = (char*)ue + sizeof(*ue); ue.elem_data_size = private_size.
- ENUMERATED: snd_ctl_elem_init_enum_names (copy from info.value.enumerated.names_ptr; up to 64 KiB; names individually < 64 bytes; matches items count).
- Set kctl.info = snd_ctl_elem_user_info (or _enum_info for ENUMERATED).
- Set kctl.get = snd_ctl_elem_user_get if READ.
- Set kctl.put = snd_ctl_elem_user_put if WRITE (memcmp + memcpy; returns change-flag).
- Set kctl.tlv.c = snd_ctl_elem_user_tlv if TLV_WRITE.
- __snd_ctl_add_replace(card, kctl, CTL_ADD_EXCLUSIVE).

REQ-25: snd_ctl_remove_user_ctl(file, id):
- Find kctl. Require vd[0].access & ACCESS_USER else -EINVAL.
- All vd[].owner must be either NULL or == file ?: -EBUSY (cannot remove others').
- snd_ctl_remove_locked.

REQ-26: snd_ctl_elem_lock/unlock(file, id):
- LOCK: vd.owner ?: -EBUSY; else vd.owner = file.
- UNLOCK: !vd.owner ?: -EINVAL; vd.owner != file ?: -EPERM; vd.owner = NULL.

REQ-27: snd_ctl_activate_id(card, id, active):
- Take controls_rwsem write.
- kctl = snd_ctl_find_id ?: -ENOENT.
- vd = kctl.vd[ioff].
- active && !(vd.access & INACTIVE): return 0 (already active).
- !active && (vd.access & INACTIVE): return 0 (already inactive).
- Toggle vd.access INACTIVE bit.
- downgrade_write; snd_ctl_notify_one(MASK_INFO, kctl, ioff); up_read.
- Return 1 (changed).

REQ-28: snd_ctl_rename(card, kctl, name) / snd_ctl_rename_id(card, src, dst):
- Take controls_rwsem write.
- remove_hash_entries (old name hash invalidated).
- strscpy new name (warn on truncation) OR replace id wholesale (rename_id: keeps original numid).
- add_hash_entries.
- Note: should only be used during card init (numid stability assumed by userspace).

REQ-29: snd_ctl_register_layer / _disconnect_layer / _request_layer:
- snd_ctl_layer singly-linked head under snd_ctl_layer_rwsem.
- register: prepend lops; iterate all existing cards under controls_rwsem read and call lops.lregister.
- disconnect: unlink (no per-card ldisconnect from this path).
- request: if module not loaded, request_module(module_name).
- Used by sound/core/control_led.c for audio-mute LED.

REQ-30: snd_ctl_register_ioctl / _compat:
- snd_kctl_ioctl list under snd_ioctl_rwsem.
- Allows hwdep, pcm, rawmidi, seq, ump etc. to extend SNDRV_CTL_IOCTL_* numbers.
- Unknown ioctl falls through to list and chains via -ENOIOCTLCMD.

REQ-31: snd_ctl_dev_register (callback for snd_device_new(SNDRV_DEV_CONTROL)):
- snd_register_device(SNDRV_DEVICE_TYPE_CONTROL, card, -1, &snd_ctl_f_ops, card, card.ctl_dev).
- ctl_dev name "controlC<n>" (set in snd_ctl_create).
- Walk all layers and call lops.lregister(card).

REQ-32: snd_ctl_create(card):
- snd_device_alloc(&card.ctl_dev).
- dev_set_name(card.ctl_dev, "controlC%d", card.number).
- snd_device_new(card, SNDRV_DEV_CONTROL, card, &ops { dev_free, dev_register, dev_disconnect }).

REQ-33: Helpers:
- snd_ctl_boolean_mono_info: type=BOOLEAN, count=1, min=0, max=1.
- snd_ctl_boolean_stereo_info: type=BOOLEAN, count=2, min=0, max=1.
- snd_ctl_enum_info(info, channels, items, names[]): type=ENUMERATED, count=channels, items=items, name[info.value.enumerated.item] copied via strscpy; warn on > sizeof(info.value.enumerated.name).

## Acceptance Criteria

- [ ] AC-1: snd_ctl_new1 from template ⟹ kctl with correct id/count/info/get/put; access masked to legal subset.
- [ ] AC-2: snd_ctl_add ⟹ kctl on card.controls; unique numid assigned; SNDRV_CTL_EVENT_MASK_ADD emitted per element.
- [ ] AC-3: snd_ctl_add of duplicate id with CTL_ADD_EXCLUSIVE ⟹ -EBUSY; instance freed.
- [ ] AC-4: snd_ctl_replace with non-existent id and add_on_replace=false ⟹ -EINVAL; instance freed.
- [ ] AC-5: snd_ctl_find_id with numid != 0 ⟹ O(1) xa_load; numid == 0 ⟹ name-hash hit or list-walk fallback.
- [ ] AC-6: ioctl ELEM_READ ⟹ kctl.get called; out-of-range integer ⟹ -EINVAL ("access overflow").
- [ ] AC-7: ioctl ELEM_WRITE with kctl.put returning >0 ⟹ SNDRV_CTL_EVENT_MASK_VALUE notification + read() event.
- [ ] AC-8: ioctl ELEM_LOCK twice from same fd ⟹ second -EBUSY (vd.owner already set).
- [ ] AC-9: ioctl ELEM_UNLOCK from non-owner ⟹ -EPERM.
- [ ] AC-10: ioctl ELEM_WRITE while owner != current fd ⟹ -EPERM.
- [ ] AC-11: ioctl ELEM_ADD with USER access ⟹ user_element with elem_data + tlv_data; alloc accounted in card.user_ctl_alloc_size.
- [ ] AC-12: ioctl TLV_READ on static tlv.p ⟹ copy_to_user header + payload bytes.
- [ ] AC-13: ioctl TLV_WRITE on element without TLV_CALLBACK ⟹ -ENXIO.
- [ ] AC-14: ioctl SUBSCRIBE_EVENTS=1, then snd_ctl_notify ⟹ subsequent read returns snd_ctl_event { type=ELEM, mask, id }.
- [ ] AC-15: poll on subscribed fd returns EPOLLIN once events queued; clears on read.
- [ ] AC-16: snd_ctl_activate_id flips INACTIVE bit, emits MASK_INFO, returns 1 if changed.
- [ ] AC-17: snd_ctl_release clears vd.owner for any vd owned by closing ctl (auto-unlock).
- [ ] AC-18: SNDRV_CTL_ELEM_TYPE_ENUMERATED with items == 0 ⟹ snd_ctl_check_elem_info rejects -EINVAL.

## Architecture

```
struct KControl {
  list: ListLink,                            // card.controls
  id: CtlElemId {
    iface: u32,                              // SNDRV_CTL_ELEM_IFACE_*
    device: u32,
    subdevice: u32,
    name: [u8; 44],
    index: u32,
    numid: u32,
  },
  count: u32,
  info: KControlInfoFn,
  get: Option<KControlGetFn>,
  put: Option<KControlPutFn>,
  tlv: TlvUnion {
    Static(&'static [u32]),                  // tlv.p: [type, length, payload...]
    Callback(KControlTlvFn),                 // tlv.c
  },
  private_value: u64,
  private_data: *mut (),
  private_free: Option<fn(&mut KControl)>,
  vd: FlexArray<KControlVolatile>,           // __counted_by(count)
}

struct KControlVolatile {
  access: u32,                               // SNDRV_CTL_ELEM_ACCESS_*
  owner: Option<*mut CtlFile>,
}

struct CtlFile {
  list: ListLink,                            // card.ctl_files
  card: *mut SndCard,
  pid: Pid,
  preferred_subdevice: [i32; SND_CTL_SUBDEV_ITEMS],
  subscribed: bool,
  events: List<KctlEvent>,
  change_sleep: WaitQueue,
  read_lock: SpinLock,
  fasync: Option<FasyncHelper>,
  user_pversion: u32,
}

struct KctlEvent {
  list: ListLink,
  id: CtlElemId,
  mask: u32,                                 // coalesced SNDRV_CTL_EVENT_MASK_*
}
```

`Ctl::new1(ncontrol, private_data) -> Option<Box<KControl>>`:
1. snd_BUG_ON(!ncontrol || !ncontrol.info) ⟹ return None.
2. count = ncontrol.count.max(1); reject > MAX_CONTROL_COUNT.
3. access = ncontrol.access; access = if access == 0 { READWRITE } else { access & ALLOWED_MASK }.
4. kctl = kzalloc_flex(KControl, vd, count); per-idx vd[idx] = { access, owner: None }.
5. kctl.id = { iface, device, subdevice, index, numid: 0 }.
6. strscpy(kctl.id.name, ncontrol.name) — warn on truncation.
7. kctl.info/get/put/tlv.p/private_value/private_data set from ncontrol.
8. Return Some(kctl).

`Ctl::add(card, kctl) -> Result<(), i32>`:
1. Snd_ctl_add_replace(card, kctl, CTL_ADD_EXCLUSIVE) (sec REQ-7).
2. On error: snd_ctl_free_one(kctl) and propagate -E*.

`Ctl::find_id(card, id) -> Option<&KControl>`:
1. id.numid != 0 ⟹ return find_numid(card, id.numid).
2. h = get_ctl_id_hash(id) — 37-multiplier on iface/device/subdevice/name/index.
3. CONFIG_SND_CTL_FAST_LOOKUP: kctl = xa_load(card.ctl_hash, h); if kctl && elem_id_matches: return Some.
4. !card.ctl_hash_collision ⟹ return None.
5. read_lock_irqsave(controls_rwlock); linear-walk card.controls; return first elem_id_matches.

`Ctl::notify(card, mask, id)`:
1. card.shutdown ⟹ return.
2. read_lock_irqsave(controls_rwlock).
3. For ctl in card.ctl_files where ctl.subscribed:
   a. spin_lock(ctl.read_lock).
   b. Coalesce: find existing ev with same numid; ev.mask |= mask ⟹ done.
   c. ev = kzalloc(KctlEvent, GFP_ATOMIC); on OOM dev_err and skip.
   d. ev.id = *id; ev.mask = mask; list_add_tail(ctl.events).
   e. wake_up(ctl.change_sleep).
   f. spin_unlock.
   g. snd_kill_fasync(ctl.fasync, SIGIO, POLL_IN).

`Ctl::elem_read(card, control) -> Result<(), i32>`:
1. Take controls_rwsem read.
2. kctl = find_id ?: -ENOENT.
3. vd = kctl.vd[ioff].
4. !(vd.access & READ) || !kctl.get ⟹ -EPERM.
5. Fill remaining bytes of control.value with 0xdeadbeef pattern.
6. result = kctl.get(kctl, control).
7. sanity_check_elem_value: range/step + pattern preserved in tail ⟹ -EINVAL "access overflow".
8. Return.

`Ctl::elem_write(card, file, control) -> Result<(), i32>`:
1. down_write(controls_rwsem).
2. kctl = find_id ?: up_write; -ENOENT.
3. vd = kctl.vd[ioff].
4. !(vd.access & WRITE) || !kctl.put || (vd.owner && vd.owner != file) ⟹ up_write; -EPERM.
5. CONFIG_SND_CTL_INPUT_VALIDATION: __snd_ctl_elem_info + sanity_check_input_values.
6. result = snd_ctl_put(card, kctl, control, vd.access).
7. result < 0: up_write; -result.
8. result > 0:
   a. downgrade_write.
   b. notify_one(card, MASK_VALUE, kctl, ioff).
   c. up_read.
9. result == 0: up_write.
10. Return 0.

`Ctl::tlv_ioctl(file, buf, op_flag) -> Result<i32, i32>`:
1. lockdep_assert_held(file.card.controls_rwsem).
2. copy_from_user(header { numid, length }).
3. header.numid == 0 ⟹ -EINVAL.
4. header.length < 2 * sizeof(u32) ⟹ -EINVAL.
5. kctl = find_numid(card, header.numid) ?: -ENOENT.
6. id = kctl.id; build ioff = header.numid - kctl.id.numid.
7. vd = kctl.vd[ioff].
8. vd.access & TLV_CALLBACK ⟹ call_tlv_handler(file, op_flag, kctl, &id, buf.tlv, header.length).
   - Check pairs[op_flag] ⊆ vd.access (READ→TLV_READ, WRITE→TLV_WRITE, CMD→TLV_COMMAND); else -ENXIO.
   - kctl.tlv.c ?: -ENXIO.
   - op_flag != READ && vd.owner && vd.owner != file ⟹ -EPERM.
   - kctl.tlv.c(kctl, op_flag, header.length, buf.tlv).
9. Else if op_flag == READ: read_tlv_buf — !(vd.access & TLV_READ) || !kctl.tlv.p ⟹ -ENXIO; copy_to_user(buf.tlv, kctl.tlv.p, 2*sizeof(u32) + kctl.tlv.p[1]).
10. Else -ENXIO.

`Ctl::read(file, buffer, count) -> ssize_t`:
1. !ctl.subscribed ⟹ -EBADFD.
2. count < sizeof(snd_ctl_event) ⟹ -EINVAL.
3. spin_lock_irq(ctl.read_lock).
4. Loop while count >= event_size:
   a. While events empty:
      - O_NONBLOCK || result > 0 ⟹ -EAGAIN/result.
      - add_wait_queue(ctl.change_sleep); TASK_INTERRUPTIBLE; spin_unlock; schedule.
      - On wake: card.shutdown ⟹ -ENODEV; signal_pending ⟹ -ERESTARTSYS; re-acquire lock.
   b. kev = list_first_entry(events).
   c. ev = snd_ctl_event { type: ELEM, data.elem: { mask: kev.mask, id: kev.id } }.
   d. list_del(kev); spin_unlock; kfree(kev).
   e. copy_to_user ⟹ -EFAULT goto out.
   f. spin_lock; buffer += event_size; count -= event_size; result += event_size.
5. Return result > 0 ? result : err.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `kctl_count_bound_by_max` | INVARIANT | per-new1: count ∈ [1, MAX_CONTROL_COUNT]; vd flex-array sized to count. |
| `kctl_access_mask_restricted` | INVARIANT | per-new1: access ⊆ ALLOWED_MASK. |
| `numid_uniqueness` | INVARIANT | per-add: assigned numid range [base, base+count-1] disjoint from any existing kctl range. |
| `controls_rwsem_held_for_add_remove` | INVARIANT | per-__snd_ctl_add_replace / __snd_ctl_remove: lockdep_assert_held_write. |
| `notify_event_alloc_atomic` | INVARIANT | per-snd_ctl_notify: kzalloc(GFP_ATOMIC) since callable from irq paths. |
| `auto_unlock_on_release` | INVARIANT | per-snd_ctl_release: for each vd where vd.owner == ctl, vd.owner ← NULL. |
| `user_ctl_alloc_size_balanced` | INVARIANT | per-elem_add + per-elem_user_free: card.user_ctl_alloc_size delta = +alloc/-alloc + ±tlv ± names. |
| `tlv_callback_perm_pairing` | INVARIANT | per-call_tlv_handler: op_flag ↔ vd.access bit (READ↔TLV_READ, WRITE↔TLV_WRITE, CMD↔TLV_COMMAND). |

### Layer 2: TLA+

`sound/core/control.tla`:
- Per-card model = sequence of (add | replace | remove | lock | unlock | read | write | notify | release).
- Properties:
  - `safety_no_two_kctls_share_numid` — per-card: any two distinct kctls have disjoint numid ranges.
  - `safety_lock_owner_unique` — per-vd: at most one ctl-file owns vd at a time.
  - `safety_write_requires_owner_or_unlocked` — per-elem_write: succeeds ⟺ !vd.owner ∨ vd.owner == file.
  - `safety_release_clears_owner` — per-close(fd): vd.owner == fd ⟹ post-condition vd.owner == None.
  - `safety_event_visible_after_notify` — per-subscribed-ctl: notify(card, mask, id) ⟹ next non-blocking read returns at least one event with overlapping mask.
  - `liveness_blocked_read_wakes_on_notify` — per-read on empty queue: notify or release wakes it (-ERESTARTSYS / event / -ENODEV).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ctl::new1` post: ret.is_some ⟹ count ≥ 1 ∧ access ⊆ ALLOWED ∧ vd.len == count | `Ctl::new1` |
| `Ctl::add` post: success ⟹ kctl ∈ card.controls ∧ numid > 0 ∧ MASK_ADD emitted per index | `Ctl::add` |
| `Ctl::remove` post: kctl ∉ card.controls ∧ MASK_REMOVE emitted ∧ private_free called | `Ctl::remove` |
| `Ctl::find_id` post: returns kctl ⟺ elem_id_matches(kctl, id) (numid==0 path) ∨ kctl.numid ≤ id.numid < kctl.numid+count | `Ctl::find_id` |
| `Ctl::elem_read` post: result == 0 ⟹ kctl.get called ∧ value range checked | `Ctl::elem_read` |
| `Ctl::elem_write` post: result == 0 ∧ put returned >0 ⟹ MASK_VALUE notification queued | `Ctl::elem_write` |
| `Ctl::elem_lock` post: vd.owner == file ∨ -EBUSY | `Ctl::elem_lock` |
| `Ctl::notify` post: per-subscribed ctl, events list contains event matching (numid, mask) | `Ctl::notify` |
| `Ctl::tlv_ioctl` post: TLV_CALLBACK ⟹ tlv.c invoked iff op-perm pair matches | `Ctl::tlv_ioctl` |

### Layer 4: Verus/Creusot functional

`Per-userspace alsactl-store/restore round-trip → snd_ctl_elem_list → per-id snd_ctl_elem_info + snd_ctl_elem_read → snd_ctl_elem_write → snd_ctl_notify(MASK_VALUE) → subscribed-fd read returns event` — semantic equivalence with Linux per-Documentation/sound/designs/control-names.rst and the libasound ctl client.

## Hardening

(Inherits row-1 features from `sound/00-overview.md` § Hardening.)

Control-core reinforcement:

- **Per-controls_rwsem write-locked for add/remove/lock/unlock** — defense against per-concurrent-add UAF (validated by `lockdep_assert_held_write`).
- **Per-controls_rwlock irqsave for notify/find_numid** — defense against per-irq-context wakeup races.
- **Per-kzalloc(GFP_ATOMIC) for snd_kctl_event in notify** — defense against per-spinlock-held sleep.
- **Per-numid hole-search bounded (100000 iter)** — defense against per-numid-exhaustion DoS.
- **Per-kctl access mask filtered against ALLOWED_MASK in new1** — defense against per-rogue-flag escalation.
- **Per-vd.owner enforced on write (must match fd)** — defense against per-cross-fd value clobber.
- **Per-vd.owner auto-cleared on snd_ctl_release** — defense against per-fd-leak permanent lock.
- **Per-snd_ctl_check_elem_info validates type / count / items** — defense against per-driver-bug overflow (max_value_counts table caps per type).
- **Per-sanity_check_elem_value with 0xdeadbeef tail pattern (CONFIG_SND_CTL_DEBUG)** — defense against per-driver-get overrunning value buffer.
- **Per-user_ctl_alloc_size caps user-defined element pool (max_user_ctl_alloc_size sysctl)** — defense against per-userspace memory exhaustion via ELEM_ADD.
- **Per-snd_ctl_elem_init_enum_names size-and-name length validation (≤64 KiB total, individual <64 B)** — defense against per-userspace huge-enum allocation.
- **Per-ioctl power-ref (snd_power_ref_and_wait)** — defense against per-suspended-card ioctl.
- **Per-input-validation (CONFIG_SND_CTL_INPUT_VALIDATION) of integer ranges before put** — defense against per-driver-trusts-userspace bug.
- **Per-numid xa_store_range plus name xa_insert collision flag** — defense against per-hash-collision missed lookup (falls back to linear).
- **Per-power-suspend ⟹ -ENOPROTOOPT (SNDRV_CTL_IOCTL_POWER)** — defense against per-userspace forcing power state.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — `snd_ctl_elem_info`, `snd_ctl_elem_value`, `snd_ctl_tlv` ioctl payloads bounded against fixed struct size; enum-name aggregates capped at 64 KiB total / 64 B per name before kmalloc.
- **PAX_KERNEXEC** — `snd_ctl_f_ops`, `snd_ctl_ioctls[]`, and `snd_kcontrol_new` `get`/`put`/`info`/`tlv` callbacks resident in RX `.text`; driver-supplied control ops resolved via kCFI-signed indirection.
- **PAX_RANDKSTACK** — kstack-offset randomization on every `/dev/snd/controlC*` ioctl entry.
- **PAX_REFCOUNT** — saturating refcount on `snd_kcontrol`, `snd_ctl_file`, and `snd_card` (control device pins card via `snd_card_file_add`); defense against userspace open/close thrash.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `snd_ctl_elem_value` user-event buffer queues and TLV scratch (prevents leak of in-flight mixer values to subsequent open).
- **PAX_UDEREF (SMAP/SMEP)** — ASM_CLAC on every control-ioctl copy_from_user; `numid` indirection via `xa_load` instead of unchecked pointer.
- **PAX_RAP / kCFI** — `snd_kcontrol_new.info/get/put/tlv` per-driver callbacks dispatched via kCFI-signed indirect calls; driver-supplied vtables verified at register.
- **GRKERNSEC_HIDESYM** — `snd_ctl_dev_register`, `snd_kcontrol_new[]`, and `snd_card->controls` symbols masked from /proc/kallsyms for non-CAP_SYSLOG.
- **GRKERNSEC_DMESG** — control-add / TLV-size-mismatch / driver-validation diagnostics restricted to CAP_SYSLOG.
- **CAP_SYS_ADMIN gating for unsafe ioctls** — `SNDRV_CTL_IOCTL_POWER`, `SNDRV_CTL_IOCTL_PRIVILEGED`, control-element ADD/REMOVE on USER-type controls gated on CAP_SYS_ADMIN; per-control `SNDRV_CTL_ELEM_ACCESS_USER` validation strict.
- **SND_CTL_INPUT_VALIDATION** — `snd_ctl_elem_write` validates integer ranges (`min..max`, `step`) against driver-declared bounds before calling `put`; defense against driver-trusts-userspace bugs.
- **Power-suspend ⟹ -ENOPROTOOPT** — defense against userspace forcing power-state via SNDRV_CTL_IOCTL_POWER while card suspended.

Per-doc rationale: `/dev/snd/controlCN` is the highest-bandwidth ALSA userspace surface — every mixer and DSP-tuning userspace probe lands here, and historically the ioctl path has been a frequent CVE source (CVE-2014-4656, CVE-2017-15265, CVE-2020-27786, CVE-2023-0266). Overlapping PAX_USERCOPY + SND_CTL_INPUT_VALIDATION + kCFI on `put/info` + saturating REFCOUNT on `snd_kcontrol` keeps the historic UAF patterns blocked at the boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- sound/core/pcm.c PCM streams (covered in `pcm.md` Tier-3)
- sound/core/rawmidi.c rawmidi (covered in `rawmidi.md` Tier-3)
- sound/core/timer.c timer interface (covered separately if expanded)
- sound/core/seq/ sequencer (covered separately if expanded)
- sound/core/control_compat.c 32-bit compat marshalling (FFI-thin)
- sound/core/control_led.c audio LED layer (uses snd_ctl_register_layer hook)
- sound/core/info.c proc-fs export (covered in `info.md` Tier-3 if expanded)
- include/sound/tlv.h DB-scale TLV helpers (used by drivers; not control-core internals)
- Implementation code
