# Tier-3: drivers/w1/{w1,w1_io,w1_netlink}.c — Dallas 1-Wire bus core (master + slave discovery + netlink)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/w1/w1-core.md
upstream-paths:
  - drivers/w1/w1.c
  - drivers/w1/w1_io.c
  - drivers/w1/w1_netlink.c
  - drivers/w1/w1_int.c
  - drivers/w1/w1_family.c
  - drivers/w1/w1_internal.h
  - drivers/w1/w1_netlink.h
  - include/linux/w1.h
-->

## Summary

The Dallas/Maxim 1-Wire protocol — a half-duplex single-data-wire bus with parasitic-power option, used for low-bandwidth sensors (DS18B20 temperature, DS2438 battery monitor, DS28E04 EEPROM, iButtons). Each peripheral holds a unique 64-bit ROM-ID `<family[8] | serial[48] | crc[8]>`. The kernel framework provides per-master kthread that performs periodic ROM-search, per-family slave driver matching, sysfs surface for direct I/O (`rw` binary attr), and an optional connector/netlink control plane (`W1_CN_BUNDLE` messages for userland-driven enumeration + raw access).

This Tier-3 covers `w1.c` (~1270 lines: bus + slave/master registration, ROM-search algorithm, kthread, attach/detach), `w1_io.c` (~450 lines: bit-level write_bit / read_bit / reset / write_block / read_block / triplet using either `touch_bit` HW abstraction or bit-banging timing), and `w1_netlink.c` (~740 lines: connector-based netlink API for userland search + bulk I/O).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct w1_master` | per-master state: bus_master ops, slave list, kthread, search lock, dev_num | `drivers::w1::Master` |
| `struct w1_bus_master` | per-master ops: read_bit/write_bit, touch_bit, read_byte/write_byte, reset_bus, search, triplet, set_pullup, data | `drivers::w1::BusMaster` |
| `struct w1_slave` | per-peripheral: master ref, family, reg_num (64-bit ROM), ttl, refcnt | `drivers::w1::Slave` |
| `struct w1_family` / `w1_family_ops` | per-family-code (`fid`) driver: groups, add/remove slave | `drivers::w1::Family` |
| `struct w1_reg_num` | packed 64-bit ROM-ID: family[8] | serial[48] | crc[8] | `drivers::w1::RegNum` |
| `w1_search(master, search_type, cb)` | ROM-search per Dallas algorithm; emits found-IDs via cb | `Master::search` |
| `w1_search_process(master, search_type)` / `_process_cb(...)` | search-then-attach orchestration | `Master::search_process` |
| `w1_attach_slave_device(master, rn)` / `_slave_detach(slave)` | per-slave registration / removal | `Master::attach_slave` / `slave_detach` |
| `w1_process(data)` | per-master kthread main loop | `Master::thread` |
| `w1_process_callbacks(dev)` | drain async_list of queued commands | `Master::process_cbs` |
| `w1_touch_bit(dev, bit)` / `w1_read_bit(dev)` / `w1_write_bit(dev, bit)` | bit cycle (60us/64us timing) | `Io::*_bit` |
| `w1_reset_bus(dev)` / `w1_reset_select_slave(slave)` | bus reset + Match ROM command | `Io::reset_bus` |
| `w1_write_block(dev, buf, len)` / `w1_read_block(dev, buf, len)` | byte-stream I/O | `Io::*_block` |
| `w1_calc_crc8(data, len)` | per-block CRC8 via 256-byte LUT | `Io::crc8` |
| `cn_netlink_send_mult(...)` / `w1_cn_callback(...)` | connector/netlink delivery + dispatch | `Netlink::*` |
| `struct w1_netlink_msg` / `w1_netlink_cmd` | netlink protocol: W1_MASTER_CMD / W1_SLAVE_CMD / W1_MASTER_ADD/REMOVE / W1_LIST_MASTERS / W1_SLAVE_ADD/REMOVE | `Netlink::Msg` |
| `w1_init_netlink()` / `w1_fini_netlink()` | connector idx CN_W1_IDX registration | `Subsystem::netlink_*` |
| `w1_max_slave_count` / `w1_max_slave_ttl` / `w1_timeout` module params | per-master search bound / TTL / interval | `Subsystem::params` |

## Compatibility contract

REQ-1: bus type `"w1"` with uevent (`w1_uevent`) emitting `MODALIAS=w1-family-0xFF` matching family code; per-slave matches via `w1_family.fid` walk.

REQ-2: Per-master MUST supply a working `read_bit` + `write_bit` OR a unified `touch_bit`; either path drives the protocol via `w1_io.c` helpers. Pure byte-level `read_byte`/`write_byte`/`reset_bus` overrides bypass bit-banging.

REQ-3: ROM-search algorithm (Dallas search per app-note 187): per-bit emit `triplet(0)+triplet(1)` (or `bus_master->triplet` if provided), branch on read-back to enumerate full 64-bit ROM tree.

REQ-4: Family-code byte (`reg_num.family`) selects `struct w1_family` driver via `w1_family_get(fid)`; on no match, slave gets registered against `w1_default_family` exposing the generic `rw` binary attribute.

REQ-5: Per-slave TTL: `w1_max_slave_ttl` (default 10) — number of consecutive searches a slave can be missing before `w1_slave_detach`.

REQ-6: Per-master automatic-search interval: `w1_timeout` (default 10s) + `w1_timeout_us` (default 0) — kthread sleeps that interval between searches.

REQ-7: Per-master `max_slave_count` (default 64) — search aborts once that many slaves observed within one cycle.

REQ-8: DS28E04 family quirk: per-spec, this part has a documented CRC inversion; `W1_FAMILY_DS28E04 = 0x1C` handled explicitly.

REQ-9: Strong-pullup support: per-master `set_pullup` op programs duration; w1 core arms before write and clears after.

REQ-10: `w1_disable_irqs` module param (default 0) — disables local IRQ for the bit-cycle to defend against jitter; trade-off documented.

REQ-11: Netlink connector protocol per `w1_netlink.h`: each request is `cn_msg + w1_netlink_msg + (w1_netlink_cmd)*`; commands include W1_CMD_READ, W1_CMD_WRITE, W1_CMD_SEARCH, W1_CMD_ALARM_SEARCH, W1_CMD_TOUCH, W1_CMD_RESET, W1_CMD_SLAVE_ADD, W1_CMD_SLAVE_REMOVE, W1_CMD_LIST_MASTERS.

REQ-12: Async-command queue: per-master `async_list` is appended to from netlink path; per-master kthread dequeues + runs cb under `dev->mutex`.

## Acceptance Criteria

- [ ] AC-1: Connect DS18B20 to GPIO-bitbang w1 master; `cat /sys/bus/w1/devices/28-.../temperature` returns valid reading.
- [ ] AC-2: Hotplug: detach DS18B20 → after `w1_max_slave_ttl * w1_timeout` seconds, sysfs node removed.
- [ ] AC-3: Family-driver match: `modprobe w1_therm` after attach triggers `w1_therm.ko` probe; before modprobe, slave reachable via default `rw` attr.
- [ ] AC-4: Netlink: userland tool issues W1_LIST_MASTERS + per-master W1_CMD_SEARCH; receives ROM list matching sysfs enumeration.
- [ ] AC-5: Bulk write via `rw` attr + read-back through W1_CMD_READ returns identical bytes.
- [ ] AC-6: Strong-pullup: DS2406 latch-write succeeds when pullup armed; fails (timeout) when unarmed on hardware requiring it.
- [ ] AC-7: ROM-search converges deterministically with simulated multi-slave topology in module-test harness.
- [ ] AC-8: CRC validation: corrupted ROM (forced CRC mismatch) rejected with `-EIO` from search path.

## Architecture

`Master` lives in `drivers::w1::Master`:

```
struct Master {
  dev: KernelDevice,
  bus_master: KBox<BusMaster>,
  slist: ListHead<Slave>,        // attached slaves
  slave_count: u32,
  slave_ttl: u32,
  ttl: u32,
  max_slave_count: u32,
  attempts: u64,
  search_count: i32,             // -1 unlimited, 0 disabled, positive countdown
  search_id: u64,                // ROM-search progress state
  mutex: Mutex<()>,
  bus_mutex: Mutex<()>,
  list_mutex: Mutex<()>,
  async_list: ListHead<AsyncCmd>,
  thread: KThreadHandle,
  refcnt: Atomic<i32>,
  enable_pullup: bool,
  pullup_duration: u32,
  flags: u64,
  name: KString,
}

struct Slave {
  name: KString,                  // family-serial canonical name
  dev: KernelDevice,
  master: Arc<Master>,
  family: Arc<Family>,
  reg_num: RegNum,                // 64-bit ROM
  ttl: i32,
  refcnt: Atomic<i32>,
  family_data: KBox<dyn Any>,
}
```

Per-master init `Master::add(master)` (w1_int.c):
1. Validate ops: at minimum (`read_bit && write_bit`) OR `touch_bit`.
2. Allocate dev_num from `w1_masters` list.
3. Init list, mutexes, set `max_slave_count = w1_max_slave_count`, `slave_ttl = w1_max_slave_ttl`.
4. `device_register(&master->dev)` under bus type `w1_bus_type`.
5. Spawn kthread `w1_process` with `name`.
6. Add to `w1_masters` under `w1_mlock`.

Kthread loop `w1_process`:
1. Compute `jtime = usecs_to_jiffies(w1_timeout * 1e6 + w1_timeout_us)`.
2. Forever:
   a. If `jremain == 0 && search_count`: lock `master->mutex`; `w1_search_process(master, W1_SEARCH)`; unlock.
   b. Lock `list_mutex`; `w1_process_callbacks(master)`; unlock.
   c. If `kthread_should_stop`: break.
   d. Sleep `jremain` (or schedule indefinitely if no active search).

ROM-search `w1_search(master, search_type, cb)`:
1. `w1_reset_bus(master)`; abort on no-presence-pulse.
2. Issue `W1_SEARCH_ROM` (0xF0) or `W1_ALARM_SEARCH` (0xEC).
3. For each of 64 bits: issue `w1_triplet`; receive (bit, complement, dir); detect conflict and pick branch; update `search_id`.
4. After 64 bits, compute CRC over collected bytes; if CRC8 fails, log + skip.
5. Invoke `cb(master, reg_num)` to register slave or refresh TTL.

Attach `w1_attach_slave_device(master, rn)`:
1. Lookup `w1_family` by `rn.family`; on miss → `w1_default_family`.
2. Allocate `struct w1_slave`; populate `reg_num`, `name = "FF-XXXXXXXXXXXX"`, `master`, `family`.
3. `__w1_attach_slave_device(slave)` registers under bus type with attribute groups + family `groups`.
4. Increment `master->slave_count`; insert into `master->slist`.

I/O helpers (`w1_io.c`):
- `w1_touch_bit(dev, bit)` — if `bus_master->touch_bit`, delegate; else bit-bang via write_bit+read_bit.
- `w1_write_bit(dev, bit)` — drive low 6us / high 64us (write-1) or low 60us / high 10us (write-0); optional local IRQ save under `w1_disable_irqs`.
- `w1_reset_bus(dev)` — 480us low + 70us release + sample presence + 410us high.
- `w1_read_block` / `w1_write_block` — byte loop using touch_bit per bit.

Netlink (`w1_netlink.c`):
- Connector ID `CN_W1_IDX`; registered via `cn_add_callback(w1_cn_callback)`.
- Each request kmalloc'd into a `w1_cb_block`: request copy + `w1_cb_node[]` per target; per-node `async_cmd` queued onto master's `async_list`.
- Reply build under `w1_cb_block`: chunked via `w1_reply_make_space` to send multiple `cn_msg`s if `W1_CN_BUNDLE` clear.
- `w1_unref_block` flushes final reply on last refcount drop.

## Hardening

(Inherits from `drivers/w1/00-overview.md` § Hardening.)

w1-core specific reinforcement:

- **Bit-cycle timing under optional local IRQ-save** — defense against soft-IRQ jitter triggering protocol violation.
- **ROM CRC8 validated after 64-bit collection** — refuses to attach slave with bad CRC.
- **Slave TTL strictly enforced** — missing slaves detached after `slave_ttl` consecutive misses.
- **Max-slave-count caps search** — defense against infinite-loop search on stuck-bus.
- **Per-master mutex serializes bus access** — kthread + netlink + sysfs writers cannot interleave bit-cycles.
- **Async-cmd queue bounded** — netlink request count limited via `w1_cb_block` refcount + per-block max size.
- **Pull-up duration bounded** — `set_pullup` refuses to arm pullup longer than per-master safety limit.
- **DS28E04 family quirk explicitly handled** — protocol corner-case documented in code, refuses generic-attr semantic for known-quirk part.
- **Slave name format `FF-XXXXXXXXXXXX` enforced** — sysfs node name validated to hex characters.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `w1_master`, `w1_slave`, `w1_family`, `w1_cb_block`, async-cmd buffer slabs; sysfs `rw` binary-attr buffers length-bounded.
- **PAX_KERNEXEC** — `w1_bus_master` ops vtable, `w1_family_ops`, `w1_bus_type` callbacks, connector callback, and per-master kthread entry live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `w1_process` (kthread), `w1_search`, `w1_attach_slave_device`, `w1_cn_callback`, and bit-cycle entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `w1_master->refcnt`, `w1_slave->refcnt`, `w1_family` refs, and `w1_cb_block.refcnt`; overflow trap defeats search/detach + netlink race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `w1_cb_block` scratch buffers, slave `family_data` payloads, async-cmd backing memory; defense against ROM/payload leak across slave-reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs / netlink / connector entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `w1_bus_master` (`read_bit`, `write_bit`, `touch_bit`, `read_byte`, `write_byte`, `reset_bus`, `search`, `triplet`, `set_pullup`), `w1_family_ops`, `w1_bus_type` callbacks, and netlink dispatch fn marked `__ro_after_init` with kCFI typed dispatch.
- **GRKERNSEC_HIDESYM** — suppress `%p` of `w1_master`, `w1_slave`, `w1_cb_block` pointers in tracepoints; gate kallsyms behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict ROM-CRC-fail, search-storm, netlink-protocol-violation banners to CAP_SYSLOG; suppress ROM-ID disclosure outside privileged readers.
- **CAP_NET_ADMIN strict** — `w1_netlink` connector requests (W1_MASTER_ADD / _REMOVE, W1_SLAVE_ADD / _REMOVE, W1_CMD_RESET, raw W1_CMD_TOUCH) require CAP_NET_ADMIN.
- **Slave registration PAX_REFCOUNT** — `w1_slave->refcnt` overflow-trapped; defense against rapid attach/detach race during search storm.
- **ROM-id length SIZE_OVERFLOW** — `w1_reg_num` parsing checks fixed 64-bit width; refuse netlink `w1_netlink_cmd` payloads that imply length > sizeof(struct w1_reg_num).
- **Async-cmd queue bounded** — per-master `async_list` length bounded; netlink request rejected with `-ENOSPC` past limit.
- **Netlink reply size bounded** — `w1_cb_block.maxlen` strict cap; refuses to build replies past declared buffer.
- **Bus-master ops null-check** — every entry validates `bus_master->*` op is non-null before call; refuses to operate with partial ops table.
- **Slave name validation** — sysfs slave names `FF-XXXXXXXXXXXX` enforced to hex digits, rejecting attacker-controlled embedded chars.

Rationale: 1-Wire devices live outside the host's normal trust boundary — they include iButton authentication tokens, EEPROM-stored secrets, and tamper-evident sensors. A loose netlink path, missing CAP gate, refcount underflow on a `w1_slave`, or unbounded async-cmd queue lets unprivileged userland (or a hot-attached rogue slave) read-write security-critical EEPROMs, soft-lockup the host via search-storm, or escalate via `w1_cb_block` UAF. RAP/kCFI on the bus-master / family ops, CAP_NET_ADMIN on the connector channel, bounded queues, ROM-ID length validation, and refcount-overflow trapping turn w1 from a hobbyist sensor bus into a structural enforcement boundary aligned with the rest of the Rookery kernel.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-master HW drivers (`drivers/w1/masters/*` — ds2490, ds2482, w1-gpio, mxc_w1, etc. — separate Tier-3s if needed)
- Per-family slave drivers (`drivers/w1/slaves/*` — w1_therm, w1_ds28e04, w1_ds2438 — separate Tier-3s if needed)
- Connector framework (`drivers/connector/cn_queue.c` — Tier-2/3 cross-ref)
- Implementation code
