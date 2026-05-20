# Tier-3: drivers/comedi/{comedi_fops,drivers,range}.c — COMEDI core (Linux Control and Measurement Device Interface)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/comedi/comedi_fops.c
  - drivers/comedi/drivers.c
  - drivers/comedi/range.c
  - drivers/comedi/comedi_internal.h
  - include/uapi/linux/comedi.h
-->

## Summary

COMEDI (Linux Control and Measurement Device Interface) is the kernel framework for data acquisition (DAQ) hardware — multi-channel ADCs/DACs, digital I/O, counter/timer cards, motion control boards. The core exposes a uniform per-device character interface `/dev/comedi<N>` plus per-subdevice character interfaces `/dev/comedi<N>_subd<M>` (when allocated) and dispatches userland ioctls + read/write/mmap to per-board low-level drivers under `drivers/comedi/drivers/`.

The core abstracts a board as a `struct comedi_device` containing N `struct comedi_subdevice` instances. A subdevice has a type (AI, AO, DI, DO, DIO, COUNTER, TIMER, MEMORY, CALIB, PROC, SERIAL, PWM), a channel count, per-channel range/aref tables, a per-subdevice async-command buffer, and an `*_insn` / `*_cmd` op-vector. Userland either issues "instructions" (per-channel synchronous reads/writes via `COMEDI_INSN` / `COMEDI_INSNLIST`) or starts a streaming "command" (`COMEDI_CMD`) that fills a ring buffer at a hardware-pacing rate consumed via `read(2)` on the per-subdevice fd.

This Tier-3 covers `comedi_fops.c` (~3700 lines: char-device, per-subdevice fops, the entire ioctl matrix), `drivers.c` (~1300 lines: per-board driver registration, attach/detach, subdevice init), and `range.c` (~130 lines: per-channel range-info ioctl).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct comedi_device` | per-board control block | `drivers::comedi::Device` |
| `struct comedi_subdevice` | per-subdevice state + ops | `drivers::comedi::Subdevice` |
| `struct comedi_driver` | per-board-driver registration record | `drivers::comedi::Driver` |
| `struct comedi_async` | per-subdevice async-command ring buffer | `drivers::comedi::Async` |
| `comedi_unlocked_ioctl(file, cmd, arg)` | main ioctl entry | `Device::ioctl` |
| `comedi_compat_ioctl(file, cmd, arg)` | 32-on-64 compat ioctl | `Device::compat_ioctl` |
| `do_devconfig_ioctl(dev, arg)` | `COMEDI_DEVCONFIG` legacy-attach | `Device::devconfig` |
| `do_bufconfig_ioctl(dev, arg)` | per-subdevice buffer alloc/resize | `Subdevice::bufconfig` |
| `do_devinfo_ioctl(...)` / `do_subdinfo_ioctl(...)` / `do_chaninfo_ioctl(...)` | introspection | `Device::devinfo` / `Subdevice::info` |
| `do_bufinfo_ioctl(...)` | get/set ring buffer head/tail | `Async::bufinfo` |
| `do_insn_ioctl(...)` / `do_insnlist_ioctl(...)` | per-instruction sync I/O | `Subdevice::insn` / `insnlist` |
| `do_cmd_ioctl(...)` / `do_cmdtest_ioctl(...)` | start / validate streaming command | `Subdevice::cmd` / `cmdtest` |
| `do_lock_ioctl(...)` / `do_unlock_ioctl(...)` | per-subdevice exclusive lock | `Subdevice::lock` / `unlock` |
| `do_cancel_ioctl(...)` / `do_poll_ioctl(...)` | stop running command / poll for samples | `Subdevice::cancel` / `poll` |
| `do_setrsubd_ioctl(...)` / `do_setwsubd_ioctl(...)` | bind fd read/write to a subdevice | `File::set_read_subd` / `set_write_subd` |
| `comedi_read(file, ubuf, n, ppos)` / `comedi_write(...)` | streaming data path | `File::read` / `write` |
| `comedi_mmap(...)` / `comedi_poll(...)` | zero-copy + poll | `File::mmap` / `poll` |
| `comedi_driver_register(drv)` / `_unregister(drv)` | per-board-driver registration | `Driver::register` / `_unregister` |
| `comedi_alloc_board_minor(hardware_device)` | allocate `/dev/comediN` minor | `Subsystem::alloc_board_minor` |
| `comedi_alloc_subdevice_minor(s)` | allocate `/dev/comediN_subdM` | `Subsystem::alloc_subdevice_minor` |
| `comedi_alloc_devpriv(dev, sz)` / `_alloc_subdevices(dev, n)` / `_alloc_spriv(s, sz)` | per-board / per-subdev priv-data alloc | `Device::alloc_*` |
| `comedi_check_chanlist(s, n, chanlist)` | validate channel+range+aref tuple list | `Subdevice::check_chanlist` |
| `comedi_get_rangeinfo` (`range.c`) | `COMEDI_RANGEINFO` ioctl | `Subdevice::rangeinfo` |
| `comedi_buf_*` (in `comedi_buf.c`, called from core) | ring-buffer accounting | `Async::buf_*` |

## Compatibility contract

REQ-1: Major char number `COMEDI_MAJOR` (98) reserved; per-device dynamic minor allocation across two regions: `[0, COMEDI_NUM_BOARD_MINORS)` for `/dev/comediN` board nodes and `[COMEDI_NUM_BOARD_MINORS, COMEDI_NUM_MINORS)` for per-subdevice nodes. `COMEDI_NUM_MINORS = 0x100`.

REQ-2: ABI `comedi.h` UAPI must remain stable: `COMEDI_VERSION_CODE`, ioctl numbers `COMEDI_DEVCONFIG..COMEDI_SETWSUBD`, structs `comedi_cmd`, `comedi_insn`, `comedi_chaninfo`, `comedi_bufconfig`, `comedi_bufinfo`, `comedi_rangeinfo`, plus subdevice/aref/range constants.

REQ-3: Per-board attach path: `comedi_driver_register` → on `COMEDI_DEVCONFIG` (or `auto_config` from PCI/USB/PCMCIA bus glue) → `comedi_driver.attach(dev, devconfig)` → driver populates `dev->n_subdevices`, each subdevice's `type`, `n_chan`, `range_table`, `insn_*`, `do_cmd`, `cancel`. Detach is `comedi_driver.detach(dev)` plus `comedi_device_detach_cleanup`.

REQ-4: Per-subdevice async-command semantics — `do_cmd_ioctl` validates `comedi_cmd` (subdev id, chanlist alloc, flags subset of `CMDF_*`, scan/convert sources within driver-supported masks), copies user chanlist via `__comedi_get_user_chanlist` (bounded 16-bit channel count), invokes `s->do_cmdtest` until stable, then `s->do_cmd`. Running command can be observed via `read`/`poll`/`mmap` and aborted via `do_cancel_ioctl`.

REQ-5: Per-subdevice ring buffer (`comedi_async`) sized by `bufconfig` ioctl bounded above by `max_bufsize` (default `CONFIG_COMEDI_DEFAULT_BUF_MAXSIZE_KB`); `CAP_SYS_ADMIN` required to raise `max_bufsize`.

REQ-6: Instruction-mode `do_insn_ioctl` / `do_insnlist_ioctl` are synchronous, channel-by-channel; per-insn `n` data words bounded by `MAX_SAMPLES = 256`; `INSN_CONFIG` lengths validated by `check_insn_config_length` against per-config-id table.

REQ-7: Per-subdevice busy/lock invariant: only one fd holds a running command on a given subdevice; `COMEDI_LOCK` further reserves the subdevice cross-fd until `COMEDI_UNLOCK` by the locking task.

REQ-8: Per-board driver attach/detach is gated by `is_device_busy` walk over all subdevices (`COMEDI_SRF_BUSY_MASK` runflag).

REQ-9: Range info: `comedi_rangeinfo` carries `(min, max, unit)` per range; min/max are 32-bit signed micro-volts/amps with `RF_EXTERNAL` flag for external-reference ranges; `range.c` validates `range_type` channel index and copies to user.

REQ-10: Streaming `read`/`write` honour `O_NONBLOCK` and `poll`'s `POLLIN`/`POLLOUT` semantics; EOF on cancel/error; `mmap` provides VM_DONTCOPY/VM_DONTEXPAND page-array view of the ring buffer (read-only for input subdevs, RW for output subdevs).

## Acceptance Criteria

- [ ] AC-1: Boot with a supported board (PCI: NI 6024E with `ni_pcimio`, USB: `ni_usb6501`); `/dev/comedi0` appears.
- [ ] AC-2: `comedi_test` userland: instruction-mode AI/AO smoke passes on every supported subdevice.
- [ ] AC-3: Streaming command on AI subdevice at 10 kSps, 1 M samples, no overrun for the default buffer size.
- [ ] AC-4: `COMEDI_BUFCONFIG` resize honours `max_bufsize`; non-root attempt to exceed it returns `-EPERM`.
- [ ] AC-5: `COMEDI_LOCK` from process A blocks `COMEDI_CMD` from process B with `-EBUSY`.
- [ ] AC-6: Concurrent `COMEDI_CANCEL` during streaming returns to idle, `read` returns short-count, `poll` reports `POLLHUP`.
- [ ] AC-7: Compat-ioctl path on x86_64 with 32-bit userland: `COMEDI_INSN` / `COMEDI_CMD` round-trip correctly.
- [ ] AC-8: Module unload of a board driver while devices attached returns `-EBUSY`.

## Architecture

`Device` lives in `drivers::comedi::Device`:

```
struct Device {
  use_count: AtomicU32,
  driver: Option<&'static Driver>,
  board_name: KStr,
  board_ptr: *const u8,        // per-driver board descriptor
  attached: AtomicBool,
  ioenabled: AtomicBool,
  spinlock: SpinLock<()>,
  mutex: Mutex<()>,
  attach_lock: RwLock<()>,
  refcount: Refcount,
  n_subdevices: u16,
  subdevices: KBox<[Subdevice]>,
  write_subdevice: Option<NonNull<Subdevice>>,
  read_subdevice:  Option<NonNull<Subdevice>>,
  private: Option<KBox<DevPriv>>,
  driver_state: AtomicU32,
  minor: u16,
  hw_dev: Option<Arc<HwDevice>>, // PCI/USB/PCMCIA parent
  class_dev: Option<NonNull<Device>>,
}

struct Subdevice {
  device: NonNull<Device>,
  index: u16,
  type_: SubdevType,
  n_chan: u32,
  subdev_flags: u32,           // SDF_READABLE / _WRITABLE / _BUSY_OWNER / _LOCK_OWNER ...
  len_chanlist: u32,
  range_table: &'static RangeTable,
  range_table_list: Option<&'static [&'static RangeTable]>,
  maxdata: u32,
  maxdata_list: Option<&'static [u32]>,
  flaglist: Option<&'static [u32]>,
  async_: Option<KBox<Async>>,
  lock_owner: AtomicPtr<File>,
  busy_owner: AtomicPtr<File>,
  runflags: AtomicU32,
  insn_read:   Option<InsnFn>,
  insn_write:  Option<InsnFn>,
  insn_bits:   Option<InsnFn>,
  insn_config: Option<InsnFn>,
  do_cmd:      Option<CmdFn>,
  do_cmdtest:  Option<CmdTestFn>,
  cancel:      Option<CancelFn>,
  poll:        Option<PollFn>,
  buf_change:  Option<BufChangeFn>,
  munge:       Option<MungeFn>,
  minor: u16,
}
```

Ioctl dispatch in `comedi_unlocked_ioctl`:

1. `comedi_file_check(file)` — validate per-fd attach generation against device; on mismatch reset `read_subdevice` / `write_subdevice` bindings.
2. Acquire `dev.mutex` for mutating ioctls (`DEVCONFIG`, `BUFCONFIG`, `LOCK`, `UNLOCK`, `CANCEL`, `POLL`, `SETRSUBD`, `SETWSUBD`); read-mostly ioctls (`DEVINFO`, `SUBDINFO`, `CHANINFO`, `RANGEINFO`, `BUFINFO`, `INSN`, `INSNLIST`, `CMD`, `CMDTEST`) take dev.attach_lock read-side only.
3. Per-ioctl handler dispatches into `do_*_ioctl` (table above); each handler validates subdevice index `< dev.n_subdevices`, channel index `< s.n_chan`, range index `< s.range_table.length`, and `aref` against `SDF_*` capability flags.

Streaming command path `do_cmd_ioctl`:
1. `__comedi_get_user_cmd` copies `comedi_cmd` from user; rejects subdev id out of range, missing `s->do_cmd`, busy subdev.
2. `__comedi_get_user_chanlist` `vmemdup_array_user` the chanlist into kernel; bounded by `s.len_chanlist` and `MAX_CHANLIST` (16 * 1024).
3. `comedi_check_chanlist(s, n, chanlist)` validates every `(chan, range, aref)` tuple.
4. `s.do_cmdtest(dev, s, &cmd)` runs to fixpoint (≤ 5 passes) to coerce/validate driver-specific timing.
5. `__comedi_set_subdevice_runflags(s, RUNNING | BUSY)`, then `s.do_cmd(dev, s)`.
6. Driver IRQ handler calls `comedi_buf_write_alloc`/`comedi_buf_write_free` to publish samples; userland `read` walks ring with `comedi_buf_read_*`.

Per-board attach `do_devconfig_ioctl`:
1. Copy `comedi_devconfig` from user; validate `board_name` length `< COMEDI_NAMELEN`.
2. Optional `aux_data` length `<= COMEDI_NUM_AUX_DATA_LEN` (firmware blob — driver-specific).
3. `comedi_device_attach(dev, &it)` (in `drivers.c`) walks the registered driver list and invokes `drv.attach(dev, &it)` on first match.
4. On success allocate per-subdevice minors via `comedi_alloc_subdevice_minor`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `subdev_index_oob` | OOB | every per-subdev ioctl rejects `subdev >= dev.n_subdevices`. |
| `chanlist_oob` | OOB | every chanlist element's chan < `s.n_chan`, range < `range_table.length`. |
| `async_buf_no_overflow` | OOB | per-subdevice ring head/tail advance bounded by `bufsize`; wrap correctly. |
| `busy_lock_excl` | RACE | at most one fd holds `BUSY` per subdev; lock_owner != null implies that fd is the only fd that may `CMD`. |

### Layer 2: TLA+

`models/comedi/async_ring.tla` — proves per-subdevice ring producer (IRQ context) / consumer (read syscall) cannot deadlock or expose stale samples across head/tail wrap.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_cmd_ioctl` post: on `Ok`, `s.runflags & RUNNING` set | `Subdevice::cmd` |
| `do_cancel_ioctl` post: `s.runflags & RUNNING` cleared and waiters woken | `Subdevice::cancel` |
| `comedi_check_chanlist` post: every tuple validated | `Subdevice::check_chanlist` |
| `attach`/`detach` symmetry: `n_subdevices` non-zero implies `subdevices` allocated and every minor freed on detach | `Device::attach`/`detach` |

### Layer 4: Verus functional

Streaming command produces same sample stream across reads regardless of buffer size (provided no overrun) — encoded as Verus equivalence over `Async::buf_read` traces.

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

comedi-core specific reinforcement:

- **DEVCONFIG firmware blob bounded** — `aux_data_length` capped at `COMEDI_NUM_AUX_DATA_LEN`; per-driver firmware signature verified before `attach`.
- **Chanlist alloc bounded** — `len_chanlist` clamped at `MAX_CHANLIST`; defense against per-CMD user-controlled vmalloc DoS.
- **INSN/INSNLIST data bounded** — per-insn `n <= MAX_SAMPLES`; insnlist count bounded by `check_insnlist_len`.
- **Ring buffer max-size policed** — raising `max_bufsize_kb` over default requires CAP_SYS_ADMIN.
- **Per-subdevice runflags atomic** — RUNNING / BUSY / ERROR / FREE_SPRIV transitions all atomic; defense against torn-flag race during CANCEL vs IRQ.
- **Minor allocation table mutex** — `comedi_board_minor_table_lock` / `_subdevice_minor_table_lock` serialize alloc/free.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `comedi_device`, `comedi_subdevice`, `comedi_async`, ring-buffer pages and chanlist arrays; `COMEDI_CMD` / `COMEDI_INSNLIST` user copies strictly size-bounded.
- **PAX_KERNEXEC** — COMEDI core text + per-board `comedi_driver` op-vectors in W^X kernel text; `__ro_after_init` for `comedi_class`, file-ops table, and ioctl dispatch table.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `comedi_unlocked_ioctl`, `do_cmd_ioctl`, `do_insn_ioctl`, and the per-driver `do_cmd` / `cancel` IRQ-context entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `comedi_device.refcount`, per-fd `comedi_file`, per-subdevice `spriv` references; overflow trap defeats attach/detach race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for chanlist arrays, async ring pages on `bufconfig` shrink, `aux_data` firmware blobs, and per-driver `devpriv` so stale ADC samples cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every `comedi_cmd` / `comedi_insn` / `comedi_chaninfo` copy; user buffers validated before `copy_from_user`/`copy_to_user`.
- **PAX_RAP / kCFI** — per-subdevice `insn_*`, `do_cmd`, `do_cmdtest`, `cancel`, `poll`, and per-driver `attach` / `detach` function pointers marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-board MMIO/IO-port disclosure behind CAP_SYSLOG; suppress `%p` in attach banners and IRQ tracepoints.
- **GRKERNSEC_DMESG** — restrict attach/detach, firmware-load-fail, and IRQ-spurious banners to CAP_SYSLOG so attackers cannot fingerprint DAQ HW via dmesg.
- **CAP_SYS_RAWIO on `/dev/comedi*`** — opening `/dev/comediN` or `/dev/comediN_subdM` requires CAP_SYS_RAWIO in the device's user namespace; raises `cap_dac_override` won't bypass.
- **`COMEDI_CMD` user copy** — PAX_USERCOPY-validated; `vmemdup_array_user` bounds chanlist and rejects oversize before allocation.
- **Subdevice id validated** — every ioctl checks `subdev < dev.n_subdevices` and `chan < s.n_chan`; defense against board-driver index trust.
- **Board firmware signature** — `aux_data` blobs delivered through `COMEDI_DEVCONFIG` checked against per-driver-allowlisted hash before driver `attach` consumes them; refuse unsigned firmware on locked-down kernels.
- **Channel-range bounds** — every `(chan, range, aref)` tuple in `comedi_check_chanlist` validated; refuse `RF_EXTERNAL` aref unless the subdevice's `SDF_DIFF` / `SDF_GROUND` / `SDF_COMMON` advertises it.
- **`COMEDI_BUFCONFIG` raise** — increasing `max_bufsize_kb` past the boot default requires CAP_SYS_ADMIN; defense against per-fd vmalloc-flood DoS.

Rationale: COMEDI boards expose raw bus-master DMA, ADC sample streams, and frequently kernel-loaded firmware blobs to userland — a corrupt chanlist or oversized firmware blob from `/dev/comediN` is enough to corrupt kernel memory or hijack DMA. CAP_SYS_RAWIO gating, signed firmware enforcement, bounded chanlists, refcount-overflow trapping, and kCFI on per-driver op-vectors turn COMEDI from "open the node, do anything" into a structurally bounded data-acquisition surface.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-board low-level drivers under `drivers/comedi/drivers/` (each future Tier-3 as needed).
- `comedi_buf.c` ring accounting internals (covered in `comedi-buf.md` future Tier-3).
- `kcomedilib/` in-kernel API for kernel-resident clients (covered in `kcomedilib.md` future Tier-3).
- Bus-glue `comedi_pci.c` / `_usb.c` / `_pcmcia.c` (each future Tier-3).
- 32-bit-only paths
- Implementation code
