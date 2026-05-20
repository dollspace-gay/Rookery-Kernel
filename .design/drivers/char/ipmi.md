# Tier-3: drivers/char/ipmi/* — IPMI subsystem (message handler, /dev/ipmiN device interface, SI/SSIF/IPMB transports)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/char/ipmi/ipmi_msghandler.c
  - drivers/char/ipmi/ipmi_devintf.c
  - drivers/char/ipmi/ipmi_si_intf.c
  - drivers/char/ipmi/ipmi_si_platform.c
  - drivers/char/ipmi/ipmi_si_pci.c
  - drivers/char/ipmi/ipmi_dmi.c
  - drivers/char/ipmi/ipmi_ssif.c
  - drivers/char/ipmi/ipmi_watchdog.c
-->

## Summary

The Intelligent Platform Management Interface (IPMI) subsystem provides userspace and in-kernel access to the Baseboard Management Controller (BMC). The msghandler core (`ipmi_msghandler.c`, ~5700 lines) is the central router: per-interface (`ipmi_smi`) message dispatch, per-user (`ipmi_user`) registration, sequence table for outstanding requests (`IPMI_IPMB_NUM_SEQ` slots), retry/timeout state machine, channel + BMC enumeration, IPMB/LAN/system-interface address routing, and panic-event delivery. The device interface (`ipmi_devintf.c`, ~900 lines) exports `/dev/ipmiN` via the `ipmi` chardev major and translates `IPMICTL_SEND_COMMAND`, `IPMICTL_RECEIVE_MSG[_TRUNC]`, `IPMICTL_REGISTER_FOR_CMD[_CHANS]`, channel/LUN address ioctls, and 32-bit-compat shims into msghandler calls. Underneath sit the System Interface transports (`ipmi_si_intf.c` over KCS / SMIC / BT bottom-halves with platform / PCI / DMI / hardcode / hotmod auto-probe), the SMBus SSIF transport (`ipmi_ssif.c`), the IPMB I2C transport (`ipmi_ipmb.c`), and consumers like `ipmi_watchdog.c` and `ipmi_poweroff.c`.

This Tier-3 covers the msghandler core plus the `/dev/ipmiN` chardev surface and the SI auto-detection plumbing — the parts directly visible to userspace and to in-kernel IPMI clients.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ipmi_smi` | per-interface control block (one per BMC) | `drivers::ipmi::Interface` |
| `struct ipmi_user` | per-open-fd or per-kernel-client handle | `drivers::ipmi::User` |
| `struct seq_table` (`IPMI_IPMB_NUM_SEQ`) | outstanding-request sequence allocator | `Interface::SeqTable` |
| `ipmi_create_user(if_num, hndl, data, &user)` | register a user against an interface | `User::create` |
| `ipmi_destroy_user(user)` | unregister + drain | `User::destroy` |
| `ipmi_request_settime(user, addr, msgid, msg, ...)` | submit one request with retry/timeout | `User::request_settime` |
| `ipmi_register_for_cmd[_chans](user, netfn, cmd, chans)` | route-in matching incoming command | `User::register_for_cmd` |
| `ipmi_smi_watcher_register(w)` | notify on SMI add/remove | `Subsystem::watch` |
| `ipmi_register_smi(handlers, send_info, dev, slave_addr)` | transport publishes a new SMI | `Subsystem::register_smi` |
| `ipmi_unregister_smi(intf)` | transport withdraws an SMI | `Subsystem::unregister_smi` |
| `ipmi_smi_msg_received(intf, msg)` | transport upcall on response | `Interface::msg_received` |
| `ipmi_alloc_smi_msg()` / `ipmi_free_smi_msg(msg)` | per-transmit message slabs | `MsgPool::alloc` / `free` |
| `ipmi_alloc_recv_msg(user)` / `ipmi_free_recv_msg(msg)` | per-receive message slabs | `MsgPool::alloc_recv` / `free_recv` |
| `__get_guid(intf)` / `__scan_channels(intf, id)` | BMC enumeration | `Interface::enumerate` |
| `ipmi_fops` (`open`/`release`/`ioctl`/`poll`/`fasync`) | chardev file_operations | `DevIntf::FileOps` |
| `IPMICTL_SEND_COMMAND[_SETTIME]` / `_RECEIVE_MSG[_TRUNC]` / `_REGISTER_FOR_CMD[_CHANS]` / `_SET_MY_CHANNEL_ADDRESS_CMD` | ioctl vocabulary | `DevIntf::ioctl_*` |
| `try_smi_init(new_smi)` / `smi_work(t)` (SI) | per-interface KCS/SMIC/BT bottom-half driver | `SI::Interface::work` |
| `ipmi_dmi_decode(dm, dummy)` | SMBIOS Type 38 IPMI device info | `SI::Dmi::decode` |

## Compatibility contract

REQ-1: Per-BMC `ipmi_smi` created exactly once at SMI registration; assigned a stable `if_num` exposed as `/dev/ipmiN` and `/sys/class/ipmi/ipmiN`.

REQ-2: BMC auto-detect probes in priority order — hardcoded module params, then ACPI (SPMI table), then SMBIOS Type 38 (`ipmi_dmi.c`), then PCI (`ipmi_si_pci.c`), then platform / OEM. The first responsive entry wins; later duplicates are skipped by I/O address match.

REQ-3: `IPMICTL_SEND_COMMAND` / `_SETTIME` carry an `ipmi_req[_settime]` whose `addr` is one of: system interface, IPMB, IPMB-direct, LAN. The msghandler validates address family and `data_len <= IPMI_MAX_MSG_LENGTH` before queueing.

REQ-4: Per-interface sequence-table allocator returns a 6-bit msgid; on response, `seq_table[msgid]` matches by `(seqid, addr)`. Stale-sequence responses are dropped without dispatch.

REQ-5: `IPMICTL_RECEIVE_MSG` returns the head of the user's recv-queue; `_TRUNC` variant truncates over-large payloads and reports the original length. Concurrent readers serialize on `priv->recv_msg_lock`.

REQ-6: `IPMICTL_REGISTER_FOR_CMD[_CHANS]` installs a per-`(netfn, cmd, chans)` filter on the user; matching incoming commands are dispatched to that user's recv-queue. Duplicate registration returns `-EBUSY`.

REQ-7: `poll()` reports `POLLIN | POLLRDNORM` when the user has at least one recv-msg; `fasync()` enables `SIGIO` on arrival.

REQ-8: 32-bit-on-64-bit ioctl compat translates `ipmi_req`/`_settime`/`_recv` via `get_compat_ipmi_*`; pointer fields are unpacked through `compat_ptr()` before reuse.

REQ-9: Panic-time delivery: `ipmi_panic_event` flushes the BMC SEL via `ipmi_request_in_rc_mode` without sleeping; respects `panic_op` module param.

REQ-10: Watchdog client (`ipmi_watchdog.c`) registers as an `ipmi_user`; per-pretimeout NMI/SMI delivery routed through msghandler.

REQ-11: SI transport bottom-half (`smi_event_handler` over `kcs_sm`/`smic_sm`/`ipmi_bt_sm` state machines) runs in kthread or tasklet; each transition is bounded by per-state timeout.

REQ-12: BMC firmware probe (`bmc_get_device_id`) gated by lockdown when LOCKDOWN_INTEGRITY_MAX is active for firmware-mutating commands (cold reset, firmware update).

## Acceptance Criteria

- [ ] AC-1: On a board with a KCS-attached BMC, `dmesg | grep ipmi_si` shows interface probed and `/dev/ipmi0` appears with major=ipmi-major, minor=0.
- [ ] AC-2: `ipmitool mc info` against `/dev/ipmi0` returns BMC device-id, firmware rev, IPMI version.
- [ ] AC-3: `ipmitool sel list` reads BMC SEL events through `IPMICTL_SEND_COMMAND` + `IPMICTL_RECEIVE_MSG`.
- [ ] AC-4: `IPMICTL_REGISTER_FOR_CMD_CHANS` on netfn=App,cmd=Get-Device-Id receives one matching incoming command and exactly one user gets the recv.
- [ ] AC-5: Sequence-table exhaustion test: queue `IPMI_IPMB_NUM_SEQ + 1` outstanding requests; the last returns `-EAGAIN`.
- [ ] AC-6: 32-bit user-space ioctl against 64-bit kernel succeeds (compat shim active).
- [ ] AC-7: Watchdog pretimeout fires `SIGIO` on a registered fd.
- [ ] AC-8: Panic test: `ipmi_panic_event` emits SEL panic record after `panic("test")`.
- [ ] AC-9: lockdown gate: with `kernel_lockdown=integrity`, firmware-update OEM cmd returns `-EPERM`.

## Architecture

`Interface` lives in `drivers::ipmi::Interface`:

```
struct Interface {
  intf_num: u32,                  // /dev/ipmiN minor
  handlers: &'static SmiHandlers, // transport ops
  send_info: NonNull<()>,         // transport private
  seq_table: [SeqEntry; IPMI_IPMB_NUM_SEQ],
  seq_lock: SpinLock,
  users: RwLock<List<Arc<User>>>,
  cmd_rcvrs: RwLock<List<CmdRcvr>>, // per-(netfn,cmd,chans) registered users
  waiting_rcv_msgs: SpinLock<List<KBox<RecvMsg>>>,
  hp_xmit_msgs: SpinLock<List<KBox<SmiMsg>>>, // panic / high-priority queue
  bmc: KBox<BmcDevice>,
  channels: [ChannelInfo; IPMI_MAX_CHANNELS],
  curr_seq: u16,
  refcount: Kref,
  shutting_down: AtomicBool,
  guid_set: AtomicBool,
  in_shutdown: AtomicBool,
}

struct User {
  refcount: Kref,
  intf: Arc<Interface>,
  hndl: &'static UserHndl,        // upcall on recv / on watcher
  handler_data: NonNull<()>,
  valid: AtomicBool,
  gets_events: AtomicBool,
}

struct FilePrivate { // /dev/ipmiN per-fd state
  user: Arc<User>,
  recv_msgs: SpinLock<List<KBox<RecvMsg>>>,
  wait: WaitQueue,
  fasync_queue: FasyncQueue,
  default_retries: i32,
  default_retry_time_ms: u32,
}
```

Send path:
1. `ipmi_ioctl(IPMICTL_SEND_COMMAND_SETTIME)` copies `ipmi_req_settime`, validates addr family, copies the `data[]` payload (size-checked).
2. `handle_send_req` → `ipmi_request_settime(user, addr, msgid, msg, supplied_smi, supplied_recv, ipmb_seq, source_address, source_lun, retries, retry_time_ms)`.
3. Msghandler allocates an SMI msg from the per-CPU pool, takes a free sequence-table slot, sets retry timer, hands to `handlers->sender(send_info, smi_msg)`.
4. Transport (KCS / SMIC / BT / SSIF / IPMB) clocks the request out byte-by-byte through its state machine.

Receive path:
1. Transport calls `ipmi_smi_msg_received(intf, smi_msg)` from IRQ or kthread.
2. `smi_work` runs `handle_new_recv_msgs` → matches `msg_data[0]` (netfn+lun) and `msg_data[1]` (seq+lun) against `seq_table`; on match, marks slot free, builds a `RecvMsg`, dispatches to the matched `User`.
3. Unsolicited commands matched against `cmd_rcvrs` list `(netfn, cmd, chans-bitmap)`; the first matching user (or all in a multi-user mode) receives.
4. `User->hndl->ipmi_recv_hndl` calls `file_receive_handler`, which appends to `priv->recv_msgs` and wakes/SIGIOs the fd.

Sequence-table semantics:
- `IPMI_IPMB_NUM_SEQ` (default 64) outstanding requests per interface.
- Per-slot `seqid` increments on reuse so a delayed response cannot bind to a fresh request.
- Retry timer per slot reissues the SMI msg up to `retries` times spaced by `retry_time_ms`.

BMC enumeration `Interface::enumerate`:
1. `Get-Device-Id` (App/0x01) — captures device-id, firmware, ipmi-version, manufacturer, product.
2. `Get-Device-Guid` (App/0x08) — captures GUID; published in sysfs.
3. `Get-Channel-Info` for each channel 0..15 — captures medium, protocol, supported sessions.

System Interface (`ipmi_si_intf.c`):
- Per-`smi_info` runs one of the three SI state machines:
  - **KCS** (`ipmi_kcs_sm.c`): 2-port (CMD + DATA) handshake — IBF/OBF/CMD/SMS_ATN bits.
  - **SMIC** (`ipmi_smic_sm.c`): 3-port handshake.
  - **BT** (`ipmi_bt_sm.c`): block-transfer with explicit length byte.
- Auto-probe order in `init_ipmi_si_drvs` → platform (`ipmi_si_platform.c`), PCI (`ipmi_si_pci.c`), DMI (`ipmi_dmi.c` reads SMBIOS Type 38), hardcoded (`ipmi_si_hardcode.c`), hotmod (`ipmi_si_hotmod.c`).
- I/O accessors: `ipmi_si_port_io.c` (inb/outb) or `ipmi_si_mem_io.c` (readb/writeb) per `addr_space`.

## Hardening

- `IPMICTL_SEND_COMMAND[_SETTIME]` validates `msg.data_len` against `IPMI_MAX_MSG_LENGTH` before copying.
- Per-interface response buffer in `RecvMsg` capped at `IPMI_MAX_MSG_LENGTH`; oversized responses truncated with `IPMICTL_RECEIVE_MSG_TRUNC` rather than overrun.
- Sequence-table slots are integers; allocator refuses out-of-range and stale `(msgid, seqid)` pairs.
- SMI msg allocation is bounded; on exhaustion, sender returns `-ENOMEM` rather than spin.
- Per-`ipmi_user` refcount (kref) prevents UAF when an fd closes while a response is in flight.
- `intf_free` (`kref_put`) waits for outstanding SMI msgs to drain before releasing transport.
- Panic-time path uses a separate `hp_xmit_msgs` queue and `run_to_completion` mode so it cannot deadlock on the same locks as the runtime path.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `ipmi_smi`, `ipmi_user`, `ipmi_recv_msg`, `ipmi_smi_msg`, and `file_private`; bound every `copy_to_user`/`copy_from_user` against `IPMI_MAX_MSG_LENGTH` and the user-supplied `msg.data_len`.
- **PAX_KERNEXEC** — msghandler core, ioctl table, and SI bottom-halves live in `__ro_after_init` text; `ipmi_user_hndl` and `smi_handlers` vtables marked `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `ipmi_ioctl`, `handle_send_req`, `handle_one_recv_msg`, and `smi_work` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `ipmi_smi.refcount`, `ipmi_user.refcount`, and per-`recv_msg` ref tracking; overflow trap defeats UAF on race between fd-close and response delivery.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `RecvMsg`, `SmiMsg`, and `ipmi_user` slabs so stale BMC payloads (potentially containing credentials, SEL records) cannot bleed across allocations.
- **PAX_UDEREF** — SMAP/PAN enforced on every `ipmi_ioctl` entry; `ipmi_req[_settime]`/`ipmi_recv`/`ipmi_cmdspec[_chans]` copied through canonical helpers only.
- **PAX_RAP / kCFI** — `ipmi_fops`, `ipmi_user_hndl`, `ipmi_smi_handlers`, and `ipmi_smi_watcher` indirect dispatch typed; runtime fixup forbidden.
- **GRKERNSEC_HIDESYM** — suppress `%p` of `ipmi_smi` / `ipmi_user` pointers in `/proc/ipmi*` and tracepoints; gate behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict SEL-record, BMC-handshake, and SI-probe banners to CAP_SYSLOG so attackers cannot fingerprint BMC firmware via dmesg.
- **CAP_SYS_ADMIN on `/dev/ipmiN`** — chardev mode default `0600`; opening it (and every ioctl path that mutates BMC state) gated on CAP_SYS_ADMIN in the owning user namespace.
- **IPMI command allowlist** — netfn/cmd pairs that mutate firmware (cold reset, fw update OEM netfns, sensor calibration writes) require CAP_SYS_RAWIO and an explicit lockdown gate (`LOCKDOWN_INTEGRITY_MAX`); read-only netfns (Get-Device-Id, Sensor reads, SEL reads) permitted at CAP_SYS_ADMIN.
- **Response-buffer bound** — `RecvMsg.data` cap enforced at allocation and again on copy-out; `IPMICTL_RECEIVE_MSG_TRUNC` is the only legal channel for over-large payloads.
- **BMC firmware probe lockdown gate** — `bmc_get_device_id` runtime queries permitted; OEM firmware-write netfns refused when `security_locked_down(LOCKDOWN_INTEGRITY_MAX)` returns nonzero.
- **Msg-handler kCFI** — every transport upcall (`ipmi_smi_msg_received`, `sender`, `request_events`, `set_run_to_completion`) is a kCFI-typed indirect call; misrouted transport plugins trap rather than executing wrong vtable.
- **Sequence-table integrity** — per-slot `(seqid, addr)` match required before recv dispatch; stale or attacker-replayed responses (e.g. delayed IPMB reply) are dropped with a rate-limited audit record.

Rationale: the BMC is a privileged out-of-band processor that observes RAM, power, and platform sensors; `/dev/ipmi*` is therefore a high-value target for both confidentiality (SEL records, sensor readings) and integrity (BMC firmware updates, watchdog disable, chassis power-control). CAP_SYS_ADMIN open, a netfn/cmd allowlist with CAP_SYS_RAWIO + lockdown for mutating commands, response-buffer bounds, refcount saturation across the user/recv-msg lifecycle, and kCFI on the transport vtables turn IPMI from "root-equals-BMC-equals-platform" into a structurally enforced policy boundary.

## Open Questions

- Should per-`/dev/ipmiN` opens carry a per-namespace policy table (allowlist) installable via a sysctl, rather than a single global allowlist?
- Per-OEM firmware-update netfn ranges (Dell iDRAC, HPE iLO, Lenovo XCC) — encode as a lockdown-gated table or refuse outright at integrity level?

## Out of Scope

- `ipmi_watchdog.c` per-pretimeout NMI plumbing (covered in `drivers/watchdog/ipmi-wdt.md` future Tier-3 if added)
- KCS BMC emulator (`kcs_bmc*.c`) — host-side IPMI used by ASPEED BMC firmware itself
- SSIF + IPMB I2C transports beyond enumeration
- Implementation code
