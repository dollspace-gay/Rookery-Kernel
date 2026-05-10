# Tier-3: drivers/hid/hid-core.c — HID core: descriptor parser + report dispatch + driver model

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/hid/00-overview.md
upstream-paths:
  - drivers/hid/hid-core.c (~3189 lines)
  - include/linux/hid.h
  - include/linux/hiddev.h
  - include/linux/hidraw.h
-->

## Summary

The **HID core** is the bus / parser / dispatch layer for all Human Interface Devices (USB, Bluetooth, I2C, SPI, Intel-ISH, AMD-SFH, Soundwire, virtual). The low-level transport driver (e.g. `usbhid`) fetches the raw report-descriptor byte stream from the device and hands it to the core via `hid_parse_report` / `hid_open_report`. The core walks the descriptor (USB-HID 1.11 § 6.2.2) one **item** at a time — Main / Global / Local / Reserved — and builds an in-kernel object graph: `hid_collection`, `hid_report` (per (type, report-id) tuple, registered in `device->report_enum[type].report_id_hash[id]`), `hid_field` (per Input/Output/Feature main item), `hid_usage` (per logical channel in a field). At runtime, the transport delivers each interrupt-report buffer via `hid_input_report(hid, type, data, size, interrupt)` → `__hid_input_report` → optional `hdrv->raw_event` (per-driver hook) → `hid_report_raw_event` → `hidraw_report_event` (if claimed) → `hid_process_report` (decode each field bit-stream into `field->new_value[]` via `hid_field_extract` / `snto32`) → `hid_process_event` (per-usage: `hdrv->event` hook + `hidinput_hid_event` for the input subsystem + `hiddev_hid_event` for the HID-dev character device). Drivers register via `__hid_register_driver(hdrv, owner, mod_name)` which sets `hdrv->driver.bus = &hid_bus_type`; the bus then matches via `hid_bus_match` → `hid_match_device` (dyn_list ∪ id_table) and probes via `__hid_device_probe` which calls `hdrv->probe` (or a default that does `hid_open_report` + `hid_hw_start(HID_CONNECT_DEFAULT)`). Critical for: keyboard / mouse / joystick / tablet / touchscreen / gamepad / sensor input, hidraw user-space pass-through, BPF-rdesc rewrite, HID-over-* generic dispatch.

This Tier-3 covers `drivers/hid/hid-core.c` (~3189 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hid_device` | per-device object | `HidDevice` |
| `struct hid_driver` | per-driver descriptor + callbacks | `HidDriver` |
| `struct hid_report` | per (type, id) report | `HidReport` |
| `struct hid_report_enum` | per-type hash table | `HidReportEnum` |
| `struct hid_field` | per-Input/Output/Feature main-item field | `HidField` |
| `struct hid_field_entry` | per-usage priority entry | `HidFieldEntry` |
| `struct hid_usage` | per-usage (page, code, hid) | `HidUsage` |
| `struct hid_collection` | per-Collection main-item | `HidCollection` |
| `struct hid_parser` | per-parse transient state | `HidParser` |
| `struct hid_global` | per-parser global-item env | `HidGlobal` |
| `struct hid_local` | per-parser local-item env | `HidLocal` |
| `hid_register_report()` | per (type, id) registration | `HidCore::register_report` |
| `hid_register_field()` | per main-item field alloc | `HidCore::register_field` |
| `hid_parser_main()` | per Main-item dispatch | `HidCore::parser_main` |
| `hid_parser_global()` | per Global-item dispatch | `HidCore::parser_global` |
| `hid_parser_local()` | per Local-item dispatch | `HidCore::parser_local` |
| `hid_parser_reserved()` | per Reserved-item dispatch | `HidCore::parser_reserved` |
| `hid_parse_collections()` | per-rdesc walk | `HidCore::parse_collections` |
| `hid_open_report()` / `hid_parse_report()` | per-rdesc entry | `HidCore::open_report` / `parse_report` |
| `hid_close_report()` | per-rdesc free | `HidCore::close_report` |
| `fetch_item()` | per-item byte fetch | `HidCore::fetch_item` |
| `item_udata()` / `item_sdata()` | per-item u32/i32 decode | `HidCore::item_udata` / `item_sdata` |
| `snto32()` / `s32ton()` | per signed-n conversion | `HidCore::snto32` / `s32ton` |
| `hid_field_extract()` | per-bitstream extract | `HidCore::field_extract` |
| `hid_output_report()` | per-report bit-pack | `HidCore::output_report` |
| `hid_input_report()` | per-transport entry | `HidCore::input_report` |
| `__hid_input_report()` | per-locked dispatch | `HidCore::input_report_inner` |
| `hid_report_raw_event()` | per-report decode | `HidCore::report_raw_event` |
| `hid_process_report()` | per-report field walk | `HidCore::process_report` |
| `hid_input_fetch_field()` | per-field bit decode | `HidCore::input_fetch_field` |
| `hid_input_var_field()` | per-variable-field dispatch | `HidCore::input_var_field` |
| `hid_input_array_field()` | per-array-field dispatch | `HidCore::input_array_field` |
| `hid_process_event()` | per-usage dispatch | `HidCore::process_event` |
| `hid_match_report()` / `hid_match_usage()` | per-driver-table match | `HidCore::match_report` / `match_usage` |
| `hid_match_device()` / `hid_match_id()` / `hid_match_one_id()` | per-(bus, group, vendor, product) match | `HidCore::match_device` / `match_id` / `match_one_id` |
| `hid_bus_match()` / `hid_bus_type` | bus-match callback + bus object | `HidCore::bus_match` |
| `hid_connect()` / `hid_disconnect()` | per-claim wire-up to hidinput / hiddev / hidraw | `HidCore::connect` / `disconnect` |
| `hid_hw_start()` / `hid_hw_stop()` | per-ll_driver start/stop | `HidCore::hw_start` / `hw_stop` |
| `hid_hw_open()` / `hid_hw_close()` | per-ll_driver event enable | `HidCore::hw_open` / `hw_close` |
| `hid_hw_request()` / `hid_hw_raw_request()` / `__hid_hw_raw_request()` | per Get/Set Report | `HidCore::hw_request` / `hw_raw_request` |
| `hid_hw_output_report()` / `__hid_hw_output_report()` | per output-report send | `HidCore::hw_output_report` |
| `hid_add_device()` / `hid_allocate_device()` / `hid_destroy_device()` | per-lifecycle | `HidCore::add_device` / `allocate_device` / `destroy_device` |
| `__hid_register_driver()` / `hid_unregister_driver()` | per-driver register | `HidCore::register_driver` / `unregister_driver` |
| `hid_device_probe()` / `__hid_device_probe()` / `hid_device_remove()` | per-driver probe/remove | `HidCore::device_probe` / `device_remove` |
| `hid_scan_report()` / `hid_scan_main()` / `hid_scan_collection()` | per-pre-probe group scan | `HidCore::scan_report` |
| `hid_setup_resolution_multiplier()` | per Resolution-Multiplier | `HidCore::setup_resolution_multiplier` |
| `new_id_store()` / `hid_free_dynids()` | per /sys/bus/hid/drivers/.../new_id | `HidCore::new_id_store` |
| `report_descriptor_read()` / `country_show()` | per-sysfs attrs | shared |
| `hid_uevent()` | per-modalias uevent | `HidCore::uevent` |

## Compatibility contract

REQ-1: `struct hid_report`:
- id: 0 .. HID_MAX_IDS-1; 0 == unnumbered report.
- type: HID_INPUT_REPORT / HID_OUTPUT_REPORT / HID_FEATURE_REPORT.
- size: in bits.
- application: top-level Collection usage at time of registration.
- device: back-pointer.
- field[HID_MAX_FIELDS]: array of fields; maxfield = count.
- field_entries: per-usage priority array for INPUT_REPORT.
- field_entry_list: priority-sorted list (descending priority).
- list: report_enum.report_list linkage.

REQ-2: `struct hid_report_enum`:
- numbered: 1 if any registered report has id ≠ 0.
- report_id_hash[HID_MAX_IDS]: direct-indexed; NULL if not registered.
- report_list: list of all reports of this type.

REQ-3: `struct hid_field`:
- index: position in report.field[].
- usage: ptr to usage array (variable-length tail).
- value: current values (s32 per logical channel).
- new_value: most-recent decoded values (memcpy'd into value after dispatch).
- usages_priorities: per-usage priority score.
- report_count / report_size / report_offset: count of logical channels, bits per channel, bit-offset within report.
- logical_minimum / logical_maximum, physical_minimum / physical_maximum, unit, unit_exponent: per HID 6.2.2.7.
- flags: HID_MAIN_ITEM_VARIABLE / _CONSTANT / _RELATIVE / _ARRAY / etc.
- maxusage: usage array size.
- application / usage_index: top-level Collection usage at field creation.
- report: back-pointer.

REQ-4: `struct hid_usage`:
- hid: full 32-bit Usage = (page << 16) | id.
- collection_index: index into device->collection.
- usage_index: index in field.usage[].
- type / code: input-subsystem evdev mapping (EV_KEY/EV_ABS/EV_REL + KEY_*/ABS_*/REL_*).

REQ-5: `struct hid_parser`:
- global: current global env (12-deep PUSH/POP stack via global_stack / global_stack_ptr).
- local: current local env (zeroed after each Main item).
- collection_stack: open-collection nesting.
- device: target hid_device.

REQ-6: `snto32(value, n)` per signed-n→s32:
- If !value || !n: 0.
- n = min(n, 32).
- sign_extend32(value, n − 1).

REQ-7: `s32ton(value, n)` per s32→signed-n:
- If !value || !n: 0.
- n = min(n, 32).
- a = value >> (n − 1).
- if a && a != −1: saturate to ±(1<<(n−1)) [−1 .. ].
- else: value & ((1<<n) − 1).

REQ-8: `fetch_item(start, end, item)`:
- Long item (0xFE prefix): read bSize bytes + tag; item.format = HID_ITEM_FORMAT_LONG.
- Short item: encoded `bSize:2 | bType:2 | bTag:4`; item.format = HID_ITEM_FORMAT_SHORT.
- Decodes 0/1/2/4-byte payload via item_udata/item_sdata helpers.

REQ-9: `hid_parser_global(parser, item)`:
- HID_GLOBAL_ITEM_TAG_PUSH: push parser.global onto global_stack (size HID_GLOBAL_STACK_SIZE).
- _POP: pop.
- _USAGE_PAGE: parser.global.usage_page = item_udata.
- _LOGICAL_MINIMUM: parser.global.logical_minimum = item_sdata.
- _LOGICAL_MAXIMUM: signed/unsigned dispatch based on logical_minimum sign.
- _PHYSICAL_MINIMUM / _MAXIMUM: same pattern.
- _UNIT_EXPONENT: handles common 4-bit two's-complement misencoding (HID-1.11 § 6.2.2.7) — if !(raw & 0xfffffff0): snto32(raw, 4); else raw.
- _UNIT: item_udata.
- _REPORT_SIZE: ≤ 256 bits; else -1.
- _REPORT_COUNT: ≤ HID_MAX_USAGES; else -1.
- _REPORT_ID: 1 .. HID_MAX_IDS-1; else -1.
- default: hid_err; return -1.

REQ-10: `hid_parser_local(parser, item)`:
- _DELIMITER: data==1 opens delimiter set, data==0 closes; only branch 0 processed; nested → -1; bogus close → -1.
- _USAGE: hid_add_usage (skip alt branches > 1).
- _USAGE_MINIMUM: parser.local.usage_minimum = data.
- _USAGE_MAXIMUM: count = data − usage_minimum; if count + usage_index ≥ HID_MAX_USAGES: clamp and warn; for n in [usage_minimum, data]: hid_add_usage(n, item.size).
- default: dbg_hid; return 0 (tolerated).

REQ-11: `hid_concatenate_last_usage_page(parser)`:
- If `local.usage_index == 0`: skip.
- usage_page = parser.global.usage_page.
- Walk parser.local.usage[i] from end backwards:
  - if local.usage_size[i] > 2: extended-32-bit usage already complete; skip.
  - if (local.usage[i] >> 16) == usage_page: break (already concatenated).
  - else: complete_usage(parser, i) — set local.usage[i] |= usage_page << 16.

REQ-12: `hid_parser_main(parser, item)`:
- hid_concatenate_last_usage_page(parser) first.
- HID_MAIN_ITEM_TAG_BEGIN_COLLECTION: open_collection(parser, data & 0xff).
- _END_COLLECTION: close_collection.
- _INPUT: hid_add_field(parser, HID_INPUT_REPORT, data).
- _OUTPUT: hid_add_field(parser, HID_OUTPUT_REPORT, data).
- _FEATURE: hid_add_field(parser, HID_FEATURE_REPORT, data).
- Reserved range: hid_warn_ratelimited.
- memset(&parser.local, 0, sizeof(parser.local)) — Local env resets after every Main item per HID 6.2.2.8.

REQ-13: `open_collection(parser, type)`:
- Push (parser.global.usage_page << 16) | first local usage (or 0) as collection.usage.
- Resize device.collection if collection_stack_ptr+1 > collection_size (double up to limit).
- Set collection.parent_idx = top-of-stack.
- Push onto collection_stack.

REQ-14: `close_collection(parser)`:
- Pop collection_stack; if underflow: -EINVAL.

REQ-15: `hid_add_usage(parser, usage, size)`:
- If usage_index ≥ HID_MAX_USAGES: -1.
- local.usage[usage_index] = usage; local.usage_size[usage_index] = size; local.collection_index[usage_index] = current top-of-collection-stack; ++usage_index.

REQ-16: `hid_add_field(parser, report_type, flags)`:
- usages = max(local.usage_index, global.report_count).
- usages = clamp(usages, 1, HID_MAX_USAGES).
- report = hid_register_report(device, report_type, global.report_id, application).
- field = hid_register_field(report, usages).
- field.flags = flags; field.report = report.
- field.report_size = global.report_size; field.report_count = global.report_count.
- field.report_offset = report.size; report.size += report_size * report_count.
- field.logical_{min,max}, physical_{min,max}, unit, unit_exponent: from parser.global.
- For each usage_index: field.usage[i].hid = local.usage[i]; field.usage[i].collection_index = local.collection_index[i]; field.usage[i].usage_index = i.
- field.maxusage = max usage count.
- field.application = collection top's application.

REQ-17: `hid_register_report(device, type, id, application)`:
- If id ≥ HID_MAX_IDS: NULL.
- If report_id_hash[id] already set: return existing.
- Allocate; if id != 0: report_enum.numbered = 1.
- Init list_add_tail report_list; field_entry_list init.

REQ-18: `hid_register_field(report, usages)`:
- If report.maxfield == HID_MAX_FIELDS: NULL.
- kvzalloc(sizeof(struct hid_field) + usages*sizeof(hid_usage) + 3*usages*sizeof(unsigned int), GFP_KERNEL).
- field.index = report.maxfield++; report.field[index] = field.
- field.usage = (hid_usage*)(field + 1).
- field.value = (s32*)(field.usage + usages).
- field.new_value = (s32*)(field.value + usages).
- field.usages_priorities = (s32*)(field.new_value + usages).
- field.report = report.

REQ-19: `hid_parse_collections(device)`:
- start = device.rdesc; end = start + device.rsize.
- parser = vzalloc(sizeof(*parser)).
- device.collection = kzalloc(sizeof(*device.collection) * HID_DEFAULT_NUM_COLLECTIONS).
- For each i: collection[i].parent_idx = −1.
- Reject 0-sized rdesc with -EINVAL.
- Loop fetch_item: dispatch via `dispatch_type[item.type]` = { parser_main, parser_global, parser_local, parser_reserved }; reject HID_ITEM_FORMAT_LONG; any return < 0: fail.
- After loop: must have consumed all bytes; collection_stack_ptr == 0; local.delimiter_depth == 0.
- hid_setup_resolution_multiplier(device).
- device.status |= HID_STAT_PARSED.

REQ-20: `hid_open_report(device)`:
- If HID_STAT_PARSED already: -EBUSY.
- start = device.bpf_rdesc; if !start: -ENODEV; size = device.bpf_rsize.
- If device.driver.report_fixup: kmemdup; invoke; second kmemdup (to detach from possibly-read-only memory).
- device.rdesc = start; device.rsize = size.
- error = hid_parse_collections(device); on fail: hid_close_report; return.

REQ-21: `hid_parse_report(hid, start, size)`:
- /* Low-level driver entry — supplies dev_rdesc + dev_rsize for hid_add_device. */
- hid.dev_rdesc = kmemdup(start, size, GFP_KERNEL).
- hid.dev_rsize = size.

REQ-22: `hid_close_report(device)`:
- For each report_type ∈ {INPUT, OUTPUT, FEATURE}:
  - For each report in report_enum[type].report_list: hid_free_report.
  - Reset report_enum.numbered = 0.
- kfree(device.rdesc) if owned (vs bpf_rdesc).
- kfree(device.collection); device.collection_size = 0; device.maxcollection = 0.
- device.status &= ~HID_STAT_PARSED.

REQ-23: `hid_free_report(report)`:
- For each field: kvfree(field).
- kfree(report.field_entries) (priority array).
- kfree(report).

REQ-24: `__extract(report, offset, n)` / `hid_field_extract(hid, report, offset, n)`:
- Little-endian bit-stream extract: byte = offset/8; bit_shift = offset % 8; mask = n < 32 ? (1<<n) − 1 : ~0.
- Loop: value |= (report[idx] >> bit_shift) << bit_nr; advance idx; recompute bits_to_copy; until n consumed.
- Public wrapper rate-limits warn-once if n > 32 and clamps to 32.

REQ-25: `__implement(report, offset, n, value)` / `implement(hid, report, offset, n, value)`:
- Validates value range vs n-bit signed-or-unsigned encoding (warn-once on overflow).
- Bit-by-bit pack into little-endian stream.

REQ-26: `hid_compute_report_size(report)`:
- ((report.size − 1) >> 3) + 1 bytes (round-up).

REQ-27: `hid_output_report(report, data)`:
- If report.id: data[0] = report.id; data++.
- For each field: __implement / implement for each usage with current field.value[i].

REQ-28: `hid_alloc_report_buf(report, gfp_flags)`:
- size = hid_compute_report_size(report) + (numbered ? 1 : 0).
- kzalloc(size, gfp_flags).

REQ-29: `hid_set_field(field, offset, value)`:
- if offset ≥ field.report_count: -EINVAL.
- s32ton range check: ensure value fits in field.report_size signed-or-unsigned bits.
- field.value[offset] = value; return 0.

REQ-30: `hid_find_field(hdev, report_type, application, usage)`:
- For each report in report_enum[report_type].report_list with matching application:
  - For each field, for each usage in field: if hid == usage: return field.
- Return NULL.

REQ-31: `hid_validate_values(hid, report_type, usage, field_index, report_counts)`:
- Find report whose application matches; verify report_counts of fields[field_index..].

REQ-32: `hid_input_report(hid, type, data, size, interrupt)`:
- `__hid_input_report(hid, type, data, size, interrupt, source=0, from_bpf=false, lock_already_taken=false)`.

REQ-33: `__hid_input_report(hid, type, data, size, interrupt, source, from_bpf, lock_already_taken)`:
- ret = down_trylock(&hid.driver_input_lock).
- If lock_already_taken && !ret: up; return -EINVAL.
- Else if !lock_already_taken && ret: return -EBUSY.
- If !hid.driver: -ENODEV.
- data = dispatch_hid_bpf_device_event(hid, type, data, &size, interrupt, source, from_bpf).
- If IS_ERR(data): goto unlock.
- If !size: -1; if debug_list: hid_dump_report.
- report = hid_get_report(report_enum, data); !report: -1.
- If hdrv.raw_event && hid_match_report(hid, report): ret = hdrv.raw_event(hid, report, data, size); < 0 → out.
- ret = hid_report_raw_event(hid, type, data, size, interrupt).
- If !lock_already_taken: up.

REQ-34: `hid_get_report(report_enum, data)`:
- If report_enum.numbered: report_id = data[0]; data++.
- Else: report_id = 0.
- Return report_enum.report_id_hash[report_id].

REQ-35: `hid_report_raw_event(hid, type, data, size, interrupt)`:
- report = hid_get_report; !report: out.
- If report_enum.numbered: cdata=data+1; csize=size-1.
- rsize = hid_compute_report_size(report); max_buffer_size = hid.ll_driver.max_buffer_size ?: HID_MAX_BUFFER_SIZE.
- Clamp rsize ≤ max_buffer_size - (numbered ? 1 : 0).
- If csize < rsize: warn-ratelimited; -EINVAL.
- HID_CLAIMED_HIDDEV && hid.hiddev_report_event: hid.hiddev_report_event(hid, report).
- HID_CLAIMED_HIDRAW: hidraw_report_event(hid, data, size); err → out.
- If claimed != HIDRAW-only && report.maxfield: hid_process_report; hdrv.report(hid, report) if set.
- HID_CLAIMED_INPUT: hidinput_report_event(hid, report).

REQ-36: `hid_process_report(hid, report, data, interrupt)`:
- For each field a in 0..report.maxfield: hid_input_fetch_field(hid, field[a], data).
- If field_entry_list non-empty (INPUT_REPORT with priority ordering):
  - For each entry: if field.flags & VARIABLE: hid_process_event(hid, field, &field.usage[entry.index], field.new_value[entry.index], interrupt).
  - Else (ARRAY): hid_input_array_field(hid, field, interrupt).
  - End: memcpy(field.value, field.new_value, sizeof(s32)*report_count) for each VARIABLE field.
- Else (FEATURE_REPORT or no ordering):
  - For each field: VARIABLE → hid_input_var_field; ARRAY → hid_input_array_field.

REQ-37: `hid_input_fetch_field(hid, field, data)`:
- value = field.new_value; memset(value, 0, count*sizeof(s32)); field.ignored = false.
- For n in 0..count:
  - value[n] = min < 0 ? snto32(hid_field_extract(hid, data, offset+n*size, size), size) : hid_field_extract(hid, data, offset+n*size, size).
  - If ARRAY && hid_array_value_is_valid(field, value[n]) && field.usage[value[n]-min].hid == HID_UP_KEYBOARD+1 (ErrorRollOver): field.ignored = true; return.

REQ-38: `hid_input_var_field(hid, field, interrupt)`:
- For n in 0..report_count: hid_process_event(hid, field, &field.usage[n], value[n], interrupt).
- memcpy(field.value, value, count*sizeof(s32)).

REQ-39: `hid_input_array_field(hid, field, interrupt)`:
- If field.ignored: return.
- For n in 0..count:
  - if valid(field, field.value[n]) && search(value, field.value[n], count) (key released): hid_process_event(hid, field, &usage[field.value[n]-min], 0, interrupt).
  - if valid(field, value[n]) && search(field.value, value[n], count) (key pressed): hid_process_event(hid, field, &usage[value[n]-min], 1, interrupt).
- memcpy(field.value, value, count*sizeof(s32)).

REQ-40: `hid_array_value_is_valid(field, value)`:
- value ≥ logical_minimum && value ≤ logical_maximum && (value − min) < maxusage.

REQ-41: `hid_process_event(hid, field, usage, value, interrupt)`:
- If !list_empty(&hid.debug_list): hid_dump_input.
- If hdrv && hdrv.event && hid_match_usage(hid, usage): ret = hdrv.event(hid, field, usage, value); if != 0: return (driver consumed).
- HID_CLAIMED_INPUT: hidinput_hid_event(hid, field, usage, value).
- HID_CLAIMED_HIDDEV && interrupt && hid.hiddev_hid_event: hiddev_hid_event(hid, field, usage, value).

REQ-42: `hid_match_report(hid, report)`:
- hdrv.report_table NULL → all (1).
- For each id until HID_TERMINATOR: HID_ANY_ID or report.type matches → 1; else 0.

REQ-43: `hid_match_usage(hid, usage)`:
- hdrv.usage_table NULL → all (1).
- For each id until HID_ANY_ID−1 terminator: (usage_hid ANY or eq) && (usage_type ANY or eq) && (usage_code ANY or eq) → 1.

REQ-44: `hid_match_one_id(hdev, id)`:
- (id.bus == HID_BUS_ANY || id.bus == hdev.bus) && (id.group == HID_GROUP_ANY || id.group == hdev.group) && (id.vendor == HID_ANY_ID || id.vendor == hdev.vendor) && (id.product == HID_ANY_ID || id.product == hdev.product).

REQ-45: `hid_match_id(hdev, id[])`:
- Iterate until id.bus == 0; first hid_match_one_id true: return.

REQ-46: `hid_match_device(hdev, hdrv)`:
- spin_lock(hdrv.dyn_lock); walk dyn_list (new_id-added): match_one_id → return.
- spin_unlock.
- Return hid_match_id(hdev, hdrv.id_table).

REQ-47: `hid_bus_match(dev, drv)`:
- `hid_match_device(to_hid_device(dev), to_hid_driver(drv)) != NULL`.

REQ-48: `hid_bus_type`:
- .name = "hid"; .dev_groups = hid_dev_groups (modalias, country, report_descriptor bin); .drv_groups = hid_drv_groups (new_id); .match = hid_bus_match; .probe = hid_device_probe; .remove = hid_device_remove; .uevent = hid_uevent.

REQ-49: `hid_uevent(dev, env)`:
- HID_ID=%04X:%08X:%08X; HID_NAME; HID_PHYS; HID_UNIQ; MODALIAS=hid:b%04Xg%04Xv%08Xp%08X; HID_FIRMWARE_VERSION=0x%04llX if set.

REQ-50: `hid_add_device(hdev)`:
- if HID_STAT_ADDED: -EBUSY.
- hdev.quirks = hid_lookup_quirk(hdev).
- if hid_ignore(hdev): -ENODEV.
- if !hdev.ll_driver.raw_request: -EINVAL.
- hdev.ll_driver.parse(hdev) (sets dev_rdesc / dev_rsize); !dev_rdesc → -ENODEV.
- hid_set_group(hdev).
- hdev.id = atomic_inc_return(&id).
- dev_set_name(&hdev.dev, "%04X:%04X:%04X.%04X", bus, vendor, product, id).
- hid_debug_register; device_add; HID_STAT_ADDED on success.

REQ-51: `hid_allocate_device()`:
- kzalloc; device_initialize; dev.release = hid_device_release; dev.bus = &hid_bus_type; device_enable_async_suspend.
- hid_close_report (init the report_enum slots).
- init_waitqueue_head(debug_wait); INIT_LIST_HEAD(debug_list); spin_lock_init(debug_list_lock); sema_init(&driver_input_lock, 1); mutex_init(&ll_open_lock); kref_init(&ref).
- hid_bpf_device_init.
- Return hdev.

REQ-52: `hid_destroy_device(hdev)`:
- hid_bpf_destroy_device.
- if HID_STAT_ADDED: device_del; hid_debug_unregister.
- hid_free_bpf_rdesc; kfree(dev_rdesc); reset sizes.
- put_device(&hdev.dev).

REQ-53: `hid_device_probe(dev)`:
- down_interruptible(&hdev.driver_input_lock); -EINTR on interrupt.
- hdev.io_started = false; clear HID_STAT_REPROBED.
- if !hdev.driver: __hid_device_probe(hdev, hdrv).
- if !io_started: up(&driver_input_lock).

REQ-54: `__hid_device_probe(hdev, hdrv)`:
- If !bpf_rsize: take dev_rdesc; hid_free_bpf_rdesc; bpf_rsize = dev_rsize; bpf_rdesc = call_hid_bpf_rdesc_fixup(hdev, dev_rdesc, &bpf_rsize); if rdesc changed: re-set_group.
- hid_check_device_match(hdev, hdrv, &id); !match → -ENODEV.
- devres_open_group; if fails -ENOMEM.
- hdev.quirks = hid_lookup_quirk; hdev.driver = hdrv.
- If hdrv.probe: hdrv.probe(hdev, id). Else default: hid_open_report → hid_hw_start(hdev, HID_CONNECT_DEFAULT).
- On fail: devres_release_group; hid_close_report; hdev.driver = NULL.

REQ-55: `hid_check_device_match(hdev, hdrv, *id)`:
- *id = hid_match_device(hdev, hdrv); !*id → false.
- if hdrv.match: return hdrv.match(hdev, hid_ignore_special_drivers).
- else: return !hid_ignore_special_drivers && !(hdev.quirks & HID_QUIRK_IGNORE_SPECIAL_DRIVER).

REQ-56: `hid_device_remove(dev)`:
- down(&driver_input_lock); io_started = false.
- if hdev.driver:
  - hdrv.remove(hdev) if set; else hid_hw_stop.
  - devres_release_group; hid_close_report; hdev.driver = NULL.
- if !io_started: up.

REQ-57: `__hid_register_driver(hdrv, owner, mod_name)`:
- hdrv.driver.name = hdrv.name; hdrv.driver.bus = &hid_bus_type; hdrv.driver.owner = owner; hdrv.driver.mod_name = mod_name.
- INIT_LIST_HEAD(&hdrv.dyn_list); spin_lock_init(&hdrv.dyn_lock).
- driver_register(&hdrv.driver).
- On success: bus_for_each_drv(__hid_bus_driver_added) — re-probe devices currently bound to hid-generic if hdrv.match would now take them.

REQ-58: `hid_unregister_driver(hdrv)`:
- driver_unregister(&hdrv.driver).
- hid_free_dynids(hdrv).
- bus_for_each_drv(__bus_removed_driver) — bus_rescan_devices (let hid-generic re-claim).

REQ-59: `new_id_store(drv, buf, count)`:
- sscanf "%x %x %x %lx" — bus, vendor, product, driver_data.
- ret < 3 → -EINVAL.
- Allocate hid_dynid; id.bus = bus; id.group = HID_GROUP_ANY; id.vendor / .product / .driver_data.
- spin_lock(&hdrv.dyn_lock); list_add_tail(&dynid.list, &hdrv.dyn_list); spin_unlock.
- driver_attach(&hdrv.driver).

REQ-60: `hid_connect(hdev, connect_mask)`:
- hid_bpf_connect_device.
- Quirks: HIDDEV_FORCE / HIDINPUT_FORCE; non-USB strips HIDDEV; hid_hiddev() expands HIDDEV_FORCE.
- HID_CONNECT_HIDINPUT: hidinput_connect; on success HID_CLAIMED_INPUT.
- HID_CONNECT_HIDDEV && hdev.hiddev_connect: HID_CLAIMED_HIDDEV.
- HID_CONNECT_HIDRAW: hidraw_connect; HID_CLAIMED_HIDRAW.
- HID_CONNECT_DRIVER: HID_CLAIMED_DRIVER.
- If !claimed && !driver.raw_event: -ENODEV (no listener).
- hid_process_ordering(hdev) — build priority lists.
- If CLAIMED_INPUT && (FF mask) && ff_init: hdev.ff_init.
- device_create_file(dev_attr_country).
- Banner: hid_info("%s: %s HID v%x.%02x %s [%s] on %s", buf, bus, version>>8, version&0xff, type, hdev.name, hdev.phys).

REQ-61: `hid_disconnect(hdev)`:
- device_remove_file(dev_attr_country).
- HID_CLAIMED_INPUT → hidinput_disconnect.
- HID_CLAIMED_HIDDEV → hdev.hiddev_disconnect.
- HID_CLAIMED_HIDRAW → hidraw_disconnect.
- claimed = 0.
- hid_bpf_disconnect_device.

REQ-62: `hid_hw_start(hdev, connect_mask)` / `hid_hw_stop(hdev)`:
- start: hdev.ll_driver.start(hdev); if connect_mask: hid_connect, undo on failure.
- stop: hid_disconnect; hdev.ll_driver.stop.

REQ-63: `hid_hw_open(hdev)` / `hid_hw_close(hdev)`:
- open: mutex_lock_killable(&ll_open_lock); if 0==ll_open_count++: hdev.ll_driver.open; rollback on fail; hdev.driver.on_hid_hw_open hook.
- close: mutex_lock(&ll_open_lock); if --ll_open_count == 0: hdev.driver.on_hid_hw_close; hdev.ll_driver.close.

REQ-64: `hid_hw_request(hdev, report, reqtype)`:
- If ll_driver.request: ll_driver.request(hdev, report, reqtype) and return.
- Else: __hid_request(hdev, report, reqtype) — pack the report and call ll_driver.raw_request.

REQ-65: `__hid_request(hid, report, reqtype)` (HID_REQ_GET_REPORT / SET_REPORT):
- buf = hid_alloc_report_buf; len = hid_compute_report_size + (numbered?1:0).
- If SET_REPORT: hid_output_report(report, buf).
- ret = hid.ll_driver.raw_request(hid, report.id, buf, len, report.type, reqtype).
- If GET_REPORT && ret > 0: hid_input_report(hid, report.type, numbered ? buf+1 : buf, ret, 0).

REQ-66: `hid_hw_raw_request(hdev, reportnum, buf, len, rtype, reqtype)`:
- __hid_hw_raw_request(hdev, reportnum, buf, len, rtype, reqtype, source=0, from_bpf=false).

REQ-67: `__hid_hw_raw_request(hdev, reportnum, buf, len, rtype, reqtype, source, from_bpf)`:
- max_buffer_size = hdev.ll_driver.max_buffer_size ?: HID_MAX_BUFFER_SIZE.
- if len < 1 || len > max_buffer_size || !buf: -EINVAL.
- ret = dispatch_hid_bpf_raw_requests(hdev, reportnum, buf, len, rtype, reqtype, source, from_bpf).
- If ret: return.
- Return hdev.ll_driver.raw_request(hdev, reportnum, buf, len, rtype, reqtype).

REQ-68: `hid_hw_output_report(hdev, buf, len)` / `__hid_hw_output_report(...)`:
- Range check (1 ≤ len ≤ max_buffer_size).
- BPF dispatch.
- If ll_driver.output_report: invoke and return.
- Else -ENOSYS.

REQ-69: `hid_driver_suspend / _reset_resume / _resume`:
- Forward to hdev.driver.suspend / .reset_resume / .resume if set (CONFIG_PM).

REQ-70: `hid_check_keys_pressed(hid)`:
- HID_CLAIMED_INPUT: walk hid.inputs; for each hidinput: any bit set in `input.key[]` → 1.

REQ-71: `hid_setup_resolution_multiplier(hdev)`:
- Walk INPUT and FEATURE reports; for each field with `usage == HID_GD_RESOLUTION_MULTIPLIER`: compute and apply effective multiplier to associated wheel fields (per Microsoft High-Resolution Scrolling spec).

REQ-72: `hid_scan_report / hid_scan_main / hid_scan_collection / hid_scan_input_usage / hid_scan_feature_usage`:
- Pre-probe walk of dev_rdesc to determine `hdev.group` (HID_GROUP_GENERIC / _MULTITOUCH / _LOGITECH_DJ_DEVICE / etc.) before driver match.

REQ-73: `hid_init / hid_exit` (module entry/exit):
- bus_register(&hid_bus_type); CONFIG_HID_BPF: hid_ops = &__hid_ops; hidraw_init; hid_debug_init.
- Exit: hid_ops = NULL; hid_debug_exit; hidraw_exit; bus_unregister; hid_quirks_exit(HID_BUS_ANY).

REQ-74: hidraw user interface (consumed via `hidraw_*` from drivers/hid/hidraw.c, the user-facing ABI):
- hidraw_report_event(hid, data, size): per-input-event hand-off when HID_CLAIMED_HIDRAW.
- hidraw_send_report and hidraw_get_report (in hidraw.c, called via hidraw fops): user ↔ hidraw_*_report ↔ hid_hw_raw_request / hid_hw_output_report ↔ ll_driver.

REQ-75: BPF hooks:
- dispatch_hid_bpf_device_event: re-write incoming raw report buffer per-attached HID-BPF program; return ERR_PTR on failure to short-circuit.
- dispatch_hid_bpf_raw_requests / _output_report: similar for Get/Set Report and Output Report.
- call_hid_bpf_rdesc_fixup: rewrite report-descriptor at probe time.

## Acceptance Criteria

- [ ] AC-1: hid_open_report on a valid USB-HID descriptor succeeds: HID_STAT_PARSED set; report_enum populated; field counts match descriptor.
- [ ] AC-2: Numbered descriptor (with REPORT_ID items): report_enum.numbered == 1; data[0] indexes report_id_hash; otherwise unnumbered hash[0] used.
- [ ] AC-3: hid_input_report decodes a variable INPUT report: field.new_value[] = signed/unsigned per logical_minimum sign; hid_process_event delivers each usage to hidinput / hiddev / hdrv.event.
- [ ] AC-4: hid_input_report on an ARRAY report (keyboard): key-release and key-press are emitted differentially (search() finds removed / added usages).
- [ ] AC-5: ErrorRollOver (HID_UP_KEYBOARD+1) detected: field.ignored = true; array report discarded.
- [ ] AC-6: hid_hw_raw_request(GET_REPORT) returns the device's current report payload through ll_driver.raw_request.
- [ ] AC-7: hid_hw_output_report sends through ll_driver.output_report; -ENOSYS if not supported.
- [ ] AC-8: hid_register_driver registers; bus_for_each_drv triggers re-probe of generic-bound devices that now match.
- [ ] AC-9: /sys/bus/hid/drivers/<drv>/new_id "%x %x %x %lx" adds a dyn_list entry; driver_attach re-binds.
- [ ] AC-10: hidraw claimed: hidraw_report_event called per incoming report.
- [ ] AC-11: hid_hw_open / hid_hw_close reference-count: ll_driver.open called once per first-opener; close on last.
- [ ] AC-12: BPF rdesc fixup: changed rdesc triggers hid_set_group re-scan.
- [ ] AC-13: VHOST_NET_F_VIRTIO_NET_HDR is **not relevant here** — this is HID core; analogous: report_fixup hook executed when driver provides one.
- [ ] AC-14: hid_destroy_device after hid_allocate_device frees all rdesc/bpf_rdesc/collection memory; kref drops to 0; hid_device_release called.
- [ ] AC-15: hid_open_report rejected if HID_STAT_PARSED already (returns -EBUSY).

## Architecture

```
struct HidParser {
  global: HidGlobal,
  local: HidLocal,
  global_stack: [HidGlobal; HID_GLOBAL_STACK_SIZE],
  global_stack_ptr: usize,
  collection_stack: Vec<u32>,
  collection_stack_ptr: usize,
  collection_stack_size: usize,
  device: *mut HidDevice,
  scan_flags: u32,                  // pre-probe scan only
}

struct HidReportEnum {
  numbered: u8,                     // 1 if any report has id ≠ 0
  report_id_hash: [Option<*mut HidReport>; HID_MAX_IDS],
  report_list: List<HidReport>,
}

struct HidReport {
  id: u32,
  type_: HidReportType,             // INPUT / OUTPUT / FEATURE
  size: u32,                        // bits
  application: u32,
  device: *mut HidDevice,
  field: [Option<*mut HidField>; HID_MAX_FIELDS],
  maxfield: u32,
  field_entries: Option<*mut HidFieldEntry>,
  field_entry_list: List<HidFieldEntry>,
  list: ListLink,
}

struct HidField {
  index: u32,
  usage: *mut HidUsage,             // tail-allocated
  value: *mut i32,
  new_value: *mut i32,
  usages_priorities: *mut i32,
  report_size: u32,
  report_count: u32,
  report_offset: u32,
  logical_minimum: i32,
  logical_maximum: i32,
  physical_minimum: i32,
  physical_maximum: i32,
  unit: u32,
  unit_exponent: i32,
  flags: u32,                       // HID_MAIN_ITEM_* bitmask
  maxusage: u32,
  application: u32,
  report: *mut HidReport,
  ignored: bool,
}

struct HidDevice {
  ll_driver: *const HidLlDriver,
  driver: *mut HidDriver,
  bus: u32, group: u32, vendor: u32, product: u32, version: u32,
  rdesc: *const u8,         // post-fixup descriptor in use
  rsize: u32,
  dev_rdesc: *const u8,     // raw descriptor from ll_driver.parse
  dev_rsize: u32,
  bpf_rdesc: *const u8,     // BPF-rewritten
  bpf_rsize: u32,
  collection: *mut HidCollection,
  collection_size: u32,
  maxcollection: u32,
  report_enum: [HidReportEnum; HID_REPORT_TYPES],
  status: u32,                      // HID_STAT_*
  quirks: u64,
  claimed: u32,                     // HID_CLAIMED_*
  driver_input_lock: Semaphore,
  ll_open_lock: Mutex,
  ll_open_count: i32,
  dev: Device,                      // struct device
  // ...
}

struct HidDriver {
  name: &'static str,
  id_table: *const HidDeviceId,
  dyn_list: List<HidDynid>,
  dyn_lock: Spinlock,
  probe: Option<fn(&mut HidDevice, &HidDeviceId) -> i32>,
  remove: Option<fn(&mut HidDevice)>,
  report_table: Option<*const HidReportId>,
  usage_table: Option<*const HidUsageId>,
  raw_event: Option<fn(&mut HidDevice, &HidReport, &[u8]) -> i32>,
  event: Option<fn(&mut HidDevice, &HidField, &HidUsage, i32) -> i32>,
  report: Option<fn(&mut HidDevice, &HidReport)>,
  report_fixup: Option<fn(&mut HidDevice, &mut [u8], &mut u32) -> *const u8>,
  match_: Option<fn(&HidDevice, bool) -> bool>,
  on_hid_hw_open: Option<fn(&mut HidDevice)>,
  on_hid_hw_close: Option<fn(&mut HidDevice)>,
  suspend / reset_resume / resume: Option<...>,  // CONFIG_PM
  driver: DeviceDriver,
}
```

`HidCore::parse_collections(device)`:
1. parser = vzalloc(...).
2. device.collection = kzalloc(HID_DEFAULT_NUM_COLLECTIONS); parent_idx = −1 each.
3. dispatch_type = [parser_main, parser_global, parser_local, parser_reserved].
4. Loop fetch_item(start, end, &item):
   - Reject LONG format.
   - dispatch_type[item.type](parser, &item); fail on negative.
5. Bytes consumed; collection_stack_ptr == 0; delimiter_depth == 0.
6. hid_setup_resolution_multiplier(device).
7. device.status |= HID_STAT_PARSED.

`HidCore::open_report(device)`:
1. !!(device.status & HID_STAT_PARSED) → -EBUSY.
2. start = device.bpf_rdesc; size = device.bpf_rsize; !start → -ENODEV.
3. if device.driver.report_fixup:
   - buf = kmemdup(start, size, GFP_KERNEL).
   - start = device.driver.report_fixup(device, buf, &size).
   - start = kmemdup(start, size, GFP_KERNEL) — detach from potentially const memory.
4. device.rdesc = start; device.rsize = size.
5. hid_parse_collections(device); on error: hid_close_report; return error.

`HidCore::input_report_inner(hid, type, data, size, interrupt, source, from_bpf, lock_already_taken)`:
1. ret = down_trylock(&hid.driver_input_lock).
2. (lock_already_taken && !ret) || (!lock_already_taken && ret) → -EINVAL / -EBUSY.
3. !hid.driver → unlock; -ENODEV.
4. data = dispatch_hid_bpf_device_event(...).
5. IS_ERR(data) → unlock; return PTR_ERR.
6. !size → unlock; -1.
7. report = hid_get_report(report_enum + type, data); !report → unlock; -1.
8. if hdrv.raw_event && hid_match_report: hdrv.raw_event(hid, report, data, size); < 0 → unlock; return.
9. hid_report_raw_event(hid, type, data, size, interrupt).
10. !lock_already_taken → up(&driver_input_lock).

`HidCore::report_raw_event(hid, type, data, size, interrupt)`:
1. report = hid_get_report(report_enum + type, data); !report → out.
2. cdata = numbered ? data + 1 : data; csize = numbered ? size − 1 : size.
3. rsize = hid_compute_report_size(report); clamp by max_buffer_size.
4. csize < rsize → warn-ratelimited; -EINVAL.
5. HID_CLAIMED_HIDDEV + hiddev_report_event → callback.
6. HID_CLAIMED_HIDRAW → hidraw_report_event(hid, data, size); err → out.
7. claimed != HIDRAW-only && report.maxfield: hid_process_report(hid, report, cdata, interrupt); hdrv.report(hid, report) if set.
8. HID_CLAIMED_INPUT → hidinput_report_event(hid, report).

`HidCore::process_report(hid, report, data, interrupt)`:
1. For each field a in 0..maxfield: hid_input_fetch_field(hid, field[a], data).
2. If field_entry_list non-empty (priority-ordered INPUT_REPORT):
   - For each entry in list (descending priority):
     - VARIABLE: hid_process_event(hid, field, &field.usage[index], field.new_value[index], interrupt).
     - ARRAY: hid_input_array_field(hid, field, interrupt).
   - End: memcpy(field.value, field.new_value) per VARIABLE field.
3. Else (FEATURE_REPORT):
   - For each field: VARIABLE → hid_input_var_field; ARRAY → hid_input_array_field.

`HidCore::process_event(hid, field, usage, value, interrupt)`:
1. debug_list non-empty → hid_dump_input.
2. hdrv.event && hid_match_usage(hid, usage) → ret = hdrv.event(...); ret != 0 → return.
3. HID_CLAIMED_INPUT → hidinput_hid_event(hid, field, usage, value).
4. HID_CLAIMED_HIDDEV && interrupt && hiddev_hid_event → callback.

`HidCore::field_extract(hid, report, offset, n) -> u32`:
1. n > 32 → warn-once; n = 32.
2. idx = offset/8; bit_shift = offset%8; bits_to_copy = 8 − bit_shift; value = 0; mask = (n < 32) ? (1<<n) − 1 : ~0u.
3. While n > 0: value |= (report[idx] >> bit_shift) << bit_nr; n -= bits_to_copy; bit_nr += bits_to_copy; bits_to_copy = 8; bit_shift = 0; ++idx.
4. Return value & mask.

`HidCore::add_device(hdev)`:
1. HID_STAT_ADDED → -EBUSY.
2. hdev.quirks = hid_lookup_quirk(hdev); hid_ignore(hdev) → -ENODEV.
3. !ll_driver.raw_request → -EINVAL.
4. ll_driver.parse(hdev) → device populates dev_rdesc / dev_rsize.
5. hid_set_group(hdev).
6. hdev.id = atomic_inc_return(&id).
7. dev_set_name; hid_debug_register; device_add; HID_STAT_ADDED on success.

`HidCore::device_probe(dev) / __device_probe(hdev, hdrv)`:
1. down_interruptible(&hdev.driver_input_lock); -EINTR.
2. io_started = false; clear HID_STAT_REPROBED.
3. if !hdev.driver:
   - If !bpf_rsize: bpf_rsize = dev_rsize; bpf_rdesc = call_hid_bpf_rdesc_fixup(...); if changed: re-set_group.
   - hid_check_device_match(hdev, hdrv, &id); !match → -ENODEV.
   - devres_open_group; hdev.quirks = hid_lookup_quirk; hdev.driver = hdrv.
   - hdrv.probe(hdev, id) if set; else hid_open_report → hid_hw_start(HID_CONNECT_DEFAULT).
   - On fail: devres_release_group; hid_close_report; hdev.driver = NULL.
4. if !io_started: up(&driver_input_lock).

`HidCore::register_driver(hdrv, owner, mod_name)`:
1. hdrv.driver.name = hdrv.name; .bus = &hid_bus_type; .owner = owner; .mod_name = mod_name.
2. INIT_LIST_HEAD(&hdrv.dyn_list); spin_lock_init(&hdrv.dyn_lock).
3. driver_register(&hdrv.driver).
4. On success: bus_for_each_drv(&hid_bus_type, NULL, NULL, __hid_bus_driver_added) → for each driver: if hdrv.match: bus_for_each_dev(__hid_bus_reprobe_drivers) — re-probe each device currently bound to hid-generic.

`HidCore::unregister_driver(hdrv)`:
1. driver_unregister(&hdrv.driver).
2. hid_free_dynids(hdrv): spin_lock(dyn_lock); list_for_each_entry_safe → kfree; spin_unlock.
3. bus_for_each_drv(__bus_removed_driver) → bus_rescan_devices(&hid_bus_type).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `parser_global_stack_bounded` | INVARIANT | per-parser_global: global_stack_ptr ∈ [0, HID_GLOBAL_STACK_SIZE]. |
| `parser_collection_stack_bounded` | INVARIANT | per-parse_collections: collection_stack_ptr ∈ [0, collection_stack_size]. |
| `parser_local_resets_after_main` | INVARIANT | per-parser_main: parser.local == 0 on return. |
| `report_id_in_range` | INVARIANT | per-register_report: 0 ≤ id < HID_MAX_IDS. |
| `report_size_within_max_buffer` | INVARIANT | per-report_raw_event: rsize ≤ max_buffer_size. |
| `report_count_bounded` | INVARIANT | per-parser_global: global.report_count ≤ HID_MAX_USAGES. |
| `report_size_bounded` | INVARIANT | per-parser_global: global.report_size ≤ 256. |
| `usage_index_bounded` | INVARIANT | per-add_usage: usage_index ≤ HID_MAX_USAGES. |
| `field_extract_no_overrun` | INVARIANT | per-field_extract: offset + n ≤ 8 * size_of_report_buf. |
| `array_field_value_validated` | INVARIANT | per-input_array_field: only valid (value−min) ∈ [0, maxusage) indices dereferenced. |
| `driver_input_lock_balanced` | INVARIANT | per-__hid_input_report: every down_trylock paired with up. |
| `ll_open_count_balanced` | INVARIANT | per-hw_open/close: ll_open_count non-negative; open() and close() pair exactly. |
| `dyn_list_under_spinlock` | INVARIANT | per-new_id / hid_match_device / hid_free_dynids: dyn_list ops hold hdrv.dyn_lock. |

### Layer 2: TLA+

`drivers/hid/hid-core.tla`:
- Per-parser state machine: Init → ItemFetch → Dispatch(Main|Global|Local|Reserved) → ItemFetch | Done.
- Per-device state machine: Allocated → Added → (Probed | Reprobed) → (Open | Closed).
- Properties:
  - `safety_parser_collection_balanced` — per-parse_collections: each BEGIN_COLLECTION paired with END_COLLECTION; collection_stack_ptr ends at 0.
  - `safety_local_env_isolated` — per-parser_main: parser.local reset between Main items.
  - `safety_report_id_unique` — per-register_report: report_id_hash[id] is either NULL or the registered report (idempotent).
  - `safety_claim_listener_present` — per-hid_connect: !hdev.claimed && !driver.raw_event ⟹ ENODEV (no orphan).
  - `safety_driver_input_lock_exclusive` — per-__hid_input_report: at most one task in critical section per hdev.
  - `liveness_per_probe_eventually_releases` — per-hid_device_probe: io_started=false ⟹ up(&driver_input_lock) eventually executed.
  - `liveness_per_register_driver_eventual_reprobe` — per-register_driver: matching generic-bound devices reprobed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `HidCore::open_report` post: device.status & HID_STAT_PARSED; device.rdesc and device.rsize valid; collection populated | `HidCore::open_report` |
| `HidCore::parse_collections` post: collection_stack_ptr == 0; delimiter_depth == 0; resolution multiplier applied | `HidCore::parse_collections` |
| `HidCore::register_report` post: report_id_hash[id] points to a valid struct; numbered set iff any id != 0 | `HidCore::register_report` |
| `HidCore::register_field` post: field.usage / .value / .new_value / .usages_priorities aligned tails | `HidCore::register_field` |
| `HidCore::field_extract` post: returns 0..(1<<min(n,32)) − 1 | `HidCore::field_extract` |
| `HidCore::input_report_inner` post: driver_input_lock released in every exit path | `HidCore::input_report_inner` |
| `HidCore::process_report` post: ARRAY-field key transitions emit one release+press pair per usage change | `HidCore::process_report` |
| `HidCore::add_device` post: HID_STAT_ADDED set iff device_add succeeded | `HidCore::add_device` |
| `HidCore::register_driver` post: hdrv.driver.bus == &hid_bus_type | `HidCore::register_driver` |
| `HidCore::__hid_request` post: GET_REPORT result re-fed via hid_input_report | `HidCore::__hid_request` |

### Layer 4: Verus/Creusot functional

`Per-rdesc walk → hid_parser_main / _global / _local → register_report / register_field → hid_input_report → hid_field_extract → hid_process_event → hidinput_hid_event` semantic equivalence: per-USB-HID 1.11 §§ 6.2.2.4 (Main Items), 6.2.2.7 (Global Items), 6.2.2.8 (Local Items), and 8 (Reports), and per-Documentation/hid/hid-protocol.rst.

`Per-numbered descriptor → data[0] = report.id → report_id_hash[id]` vs `Per-unnumbered → report_id_hash[0]`: bit-exact wire equivalence.

`Per-array INPUT_REPORT differential emit (release-then-press per usage transition)` vs `Per-variable INPUT_REPORT direct value emit`: equivalence vs HID-1.11 § 6.2.2.5.

`Per-driver register → bus_for_each_drv → __hid_bus_reprobe_drivers → hid-generic releases device → __hid_device_probe with new hdrv` ordering: semantic equivalence vs driver-core docs.

## Hardening

(Inherits row-1 features from `drivers/hid/00-overview.md` § Hardening.)

HID-core reinforcement:

- **Per-rdesc rsize/report_size/report_count bounded checks** — defense against per-malicious-descriptor unbounded allocation.
- **Per-collection-stack overflow → re-allocate up to bounded `HID_DEFAULT_NUM_COLLECTIONS << k` cap** — defense against per-DoS deep nesting.
- **Per-global PUSH/POP stack-overflow / underflow rejected** — defense against per-descriptor stack corruption.
- **Per-delimiter only branch 0 processed** — defense against per-ambiguous alternate-usage interpretation.
- **Per-field_extract clamp n ≤ 32 and warn-once** — defense against per-undefined-shift-behavior.
- **Per-hid_array_value_is_valid bound on usage[] index** — defense against per-OOB-usage-access from array reports.
- **Per-driver_input_lock semaphore (down_trylock)** — defense against per-reentrant input from same transport.
- **Per-numbered/unnumbered report path strict (data[0] = id vs none)** — defense against per-mismatched-length truncation.
- **Per-rsize ≤ ll_driver.max_buffer_size** — defense against per-transport buffer overflow.
- **Per-hidraw report_event respects HID_CLAIMED_HIDRAW only** — defense against per-leak of raw report to non-claimed listener.
- **Per-bpf rdesc fixup return validated; hid_set_group re-run if rdesc changed** — defense against per-stale-group misclaim.
- **Per-quirk lookup at probe (not cached across re-probe)** — defense against per-quirk-bit drift.
- **Per-hid_check_device_match honors hid_ignore_special_drivers / HID_QUIRK_IGNORE_SPECIAL_DRIVER** — defense against per-malicious-special-driver attach.
- **Per-dyn_list spinlock-protected** — defense against per-new_id race with hid_match_device.
- **Per-ll_open_count non-negative invariant** — defense against per-mismatched hid_hw_open/close.
- **Per-hid_destroy_device free path tolerates partial init** — defense against per-mid-probe-failure leak.
- **Per-hid_close_report idempotent** — defense against per-double-free on probe rollback.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/hid/hid-input.c (hidinput_connect / hidinput_hid_event / hidinput_report_event evdev mapping) — covered in `drivers/hid/hid-input.md` Tier-3.
- drivers/hid/hidraw.c (hidraw_connect / hidraw_send_report / hidraw_get_report / hidraw fops / hidraw_init/exit / minor allocation) — covered in `drivers/hid/hidraw.md` Tier-3.
- drivers/hid/hid-debug.c (hid_dump_*, hid_debug_register / _init / _exit) — covered separately if expanded.
- drivers/hid/hid-quirks.c (hid_lookup_quirk, hid_ignore, hid_quirks_exit) — covered in `drivers/hid/hid-quirks.md` Tier-3.
- drivers/hid/bpf/* (HID-BPF: dispatch_hid_bpf_*, call_hid_bpf_rdesc_fixup) — covered in `drivers/hid/hid-bpf.md` Tier-3.
- drivers/hid/usbhid/* (USB low-level driver = ll_driver.parse / .raw_request / .output_report / .start / .stop / .open / .close) — covered in `drivers/hid/usbhid.md` Tier-3.
- drivers/hid/i2c-hid/*, intel-ish-hid/*, hid-bluetooth/* (other transports) — covered separately.
- input subsystem (drivers/input/) — covered in input Tier-3 set.
- Implementation code.
