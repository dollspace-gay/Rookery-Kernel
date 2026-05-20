# Tier-3: drivers/pps/{pps,kapi,sysfs}.c — PPS framework (Pulse-Per-Second / RFC 2783 LinuxPPS)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/pps/pps.c
  - drivers/pps/kapi.c
  - drivers/pps/sysfs.c
  - drivers/pps/kc.c
  - include/linux/pps_kernel.h
  - include/uapi/linux/pps.h
-->

## Summary

The PPS (Pulse-Per-Second) framework — kernel implementation of RFC 2783 LinuxPPS. Provides a generic timestamping pipeline from a hardware PPS source (UART DCD/CTS via line-discipline, GPIO via `pps-gpio`, parallel-port via `pps-parport`, PHC via `pps-phc`, GNSS receiver) to userspace consumers via per-source `/dev/pps<id>` char device and to the in-kernel hardpps disciplining via `pps_kc_event`. The framework owns: per-source `pps_device` registration through `pps_register_source` / `pps_unregister_source`, IRQ-driven event capture via `pps_event`, per-source ioctl surface (`PPS_GETPARAMS`/`PPS_SETPARAMS`/`PPS_GETCAP`/`PPS_FETCH`/`PPS_KC_BIND`) including 32-bit compat, sysfs per-source attributes (`assert`/`clear`/`mode`/`echo`/`name`/`path`), and per-source idr indexing into a chrdev major.

This Tier-3 covers `pps.c` (~503 lines: chrdev fops, ioctls, compat ioctl, idr-managed source registration, class init, pps_lookup_dev cookie API), `kapi.c` (~219 lines: `pps_register_source`/`pps_unregister_source`, `pps_event` IRQ-side capture with assert/clear sequence counters + per-mode offset addition), and `sysfs.c` (~99 lines: per-source attribute group rendering `pps_ktime` + sequence number + mode bitmask).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pps_device` | per-source state (info, params, sequence, assert_tu/clear_tu, last_ev, queue, id, dev, async_queue, lock) | `drivers::pps::PpsDevice` |
| `struct pps_source_info` | per-source descriptor (name, path, mode, echo, owner, dev) | `drivers::pps::SourceInfo` |
| `struct pps_event_time` | dual-clock event timestamp (ts_real + ts_raw under CONFIG_NTP_PPS) | `drivers::pps::EventTime` |
| `struct pps_kparams` | per-source parameters (api_version, mode, assert_off_tu, clear_off_tu) | `drivers::pps::KParams` |
| `struct pps_ktime` | { sec, nsec, flags } pair | `drivers::pps::KTime` |
| `pps_register_source(info, default_params)` | per-source kapi register | `PpsDevice::register_source` |
| `pps_unregister_source(pps)` | per-source kapi unregister | `PpsDevice::unregister_source` |
| `pps_event(pps, ts, event, data)` | IRQ-callable event capture + assert/clear seq bump + queue wake | `PpsDevice::event` |
| `pps_lookup_dev(cookie)` | look up pps_device by per-source magic cookie | `PpsDevice::lookup_dev` |
| `pps_register_cdev(pps)` / `pps_unregister_cdev(pps)` | per-source chrdev install/remove + idr alloc | `PpsDevice::register_cdev` / `_unregister_cdev` |
| `pps_cdev_ioctl(file, cmd, arg)` | ioctl dispatch | `PpsDevice::cdev_ioctl` |
| `pps_cdev_compat_ioctl(file, cmd, arg)` | 32-bit compat | `PpsDevice::cdev_compat_ioctl` |
| `pps_cdev_pps_fetch(pps, fdata)` | wait_event_interruptible{_timeout} for next event | `PpsDevice::cdev_pps_fetch` |
| `pps_cdev_poll(file, wait)` | EPOLLIN when last_fetched_ev != last_ev | `PpsDevice::cdev_poll` |
| `pps_cdev_fasync(fd, file, on)` | SIGIO subscription | `PpsDevice::cdev_fasync` |
| `pps_kc_bind(pps, bind_args)` / `pps_kc_event(pps, ts, event)` / `pps_kc_remove(pps)` | kernel consumer (hardpps) bind/dispatch/release | `PpsDevice::kc_*` |
| `pps_groups[]` (assert/clear/mode/echo/name/path) | per-source sysfs attribute group | `PpsDevice::sysfs_groups` |
| `PPS_MAX_SOURCES` (=128) | per-system source ceiling (idr range) | `consts::PPS_MAX_SOURCES` |
| `PPS_API_VERS` | API version stamped into `params.api_version` | `consts::PPS_API_VERS` |

## Compatibility contract

REQ-1: Per-system source ceiling — `idr_alloc(&pps_idr, 0, PPS_MAX_SOURCES, GFP_KERNEL)` allocates per-source id; -ENOSPC → -EBUSY surfaced to caller.

REQ-2: Per-source character device — major `pps_major` (dynamic via `__register_chrdev(0, 0, PPS_MAX_SOURCES, "pps", ...)`); minor == source id; node `/dev/pps<id>`.

REQ-3: Source mode bitmask — `PPS_CAPTUREASSERT | PPS_CAPTURECLEAR | PPS_OFFSETASSERT | PPS_OFFSETCLEAR | PPS_CANWAIT | PPS_ECHOASSERT | PPS_ECHOCLEAR | PPS_TSFMT_TSPEC | PPS_TSFMT_NTPFP`; per-source `info.mode` is the capability mask; `params.mode` must be a subset.

REQ-4: Per-source event semantics — `pps_event(pps, ts, event, data)` called from IRQ; under `pps->lock` (irqsave): adds per-mode offset to `ts_real`, updates `assert_tu`/`clear_tu` + `assert_sequence`/`clear_sequence`, bumps `last_ev`, calls echo if subscribed, dispatches `pps_kc_event`, wakes queue, fires SIGIO via fasync.

REQ-5: `PPS_SETPARAMS` requires CAP_SYS_TIME; rejects mode without `PPS_CAPTUREASSERT|PPS_CAPTURECLEAR`; rejects bits not in `info.mode`; clears `assert_off_tu.flags`/`clear_off_tu.flags` to defeat info-leak via `PPS_GETPARAMS`.

REQ-6: `PPS_KC_BIND` requires CAP_SYS_TIME; only `PPS_KC_HARDPPS` consumer accepted; only `PPS_TSFMT_TSPEC` format; only `PPS_CAPTUREBOTH` edge.

REQ-7: `PPS_FETCH` semantics — caller-supplied timeout via `pps_fdata.timeout` (`PPS_TIME_INVALID` flag → wait forever); on event, returns `assert/clear_sequence + assert/clear_tu + current_mode`; -ETIMEDOUT after timeout; -EINTR on signal.

REQ-8: Compat ioctl handles `pps_fdata_compat` (32-bit-time variant); explicit copy of timeout and result info; falls back to native ioctl for other cmds.

REQ-9: Per-source default-params constraint — `(info.mode & default_params) == default_params` (every default bit must be a capability); per-mode unspecified-format rejected.

REQ-10: Per-source echo function — if `info.mode & (PPS_ECHOASSERT|PPS_ECHOCLEAR)` and `info.echo == NULL`, framework substitutes `pps_echo_client_default` (dev_info on each event).

REQ-11: `pps_lookup_dev(cookie)` — kludge for serial line-discipline (`pps-ldisc`); RCU-walk pps_idr by `lookup_cookie` (filled by client after `pps_register_source`).

REQ-12: Sysfs attributes — `assert`/`clear` print `%lld.%09d#%d` with sequence number; `mode` prints `%4x` mask; `echo`/`name`/`path` simple strings.

REQ-13: Lifecycle — `pps_register_source` calls `pps_register_cdev` which gets an idr slot, registers device under `pps_class`, takes a per-device get_device to keep alive until `pps_unregister_cdev` drops it; per-device free via `pps_device_destruct`.

REQ-14: `pps_unregister_source` clears `pps->lookup_cookie` (NULL) then calls `pps_kc_remove` + `pps_unregister_cdev` so no stale cookie can be used after teardown.

## Acceptance Criteria

- [ ] AC-1: Boot with `pps-gpio` configured for a GPIO pin attached to a 1 PPS source; `/dev/pps0` exists, `/sys/class/pps/pps0/` lists attributes.
- [ ] AC-2: `ppstest /dev/pps0` reports assert events every ~1 s with sequence number monotonically increasing.
- [ ] AC-3: `ppstest -k` requires CAP_SYS_TIME; non-root fails with EPERM on `PPS_KC_BIND`.
- [ ] AC-4: `PPS_SETPARAMS` with mode==0 returns -EINVAL.
- [ ] AC-5: `PPS_FETCH` with explicit short timeout returns -ETIMEDOUT when no event arrives.
- [ ] AC-6: 32-bit userland on 64-bit kernel uses `PPS_FETCH` compat path; timestamps survive round-trip.
- [ ] AC-7: `pps_unregister_source` succeeds while an open fd is held; subsequent ioctl on the stale fd returns failure cleanly without UAF.

## Architecture

```
struct PpsDevice {
  info: SourceInfo,                     // name, path, mode, echo, owner, dev
  params: KParams,                      // api_version, mode, assert_off_tu, clear_off_tu
  assert_sequence: u32, clear_sequence: u32,
  assert_tu: KTime, clear_tu: KTime,
  current_mode: i32,
  last_ev: u32, last_fetched_ev: u32,
  queue: WaitQueueHead,
  id: u32,                              // idr slot, minor == id
  lookup_cookie: *const (),             // pps-ldisc kludge
  dev: Device,                          // class = pps_class
  async_queue: *mut FasyncStruct,
  lock: SpinLock<()>,                   // IRQ-safe
}

static pps_idr:      Idr<*mut PpsDevice>;
static pps_idr_lock: Mutex<()>;          // wraps idr alloc/remove
static pps_class:    DeviceClass = "pps" with dev_groups = pps_groups;
static pps_major:    i32;                // dyn-allocated chrdev major
```

Per-source register `pps_register_source(info, default_params)`:
1. Sanity: `(info.mode & default_params) == default_params`; format bit set.
2. `kzalloc(struct pps_device)`.
3. Init `params.api_version`, `params.mode = default_params`, `info = *info`.
4. If echo-capable but no echo, substitute default.
5. `init_waitqueue_head` + `spin_lock_init`.
6. `pps_register_cdev` — idr_alloc → id; device_register under pps_class; minor = id; release = `pps_device_destruct` (kfree).

Per-source IRQ event `pps_event(pps, ts, event, data)`:
1. `BUG_ON((event & (CAPTUREASSERT|CAPTURECLEAR)) == 0)`.
2. `timespec_to_pps_ktime(&ts_real, ts->ts_real)`.
3. Take `pps->lock` irqsave.
4. If echo mode + event matches: call `info.echo(pps, event, data)`.
5. Update `current_mode = params.mode`.
6. For each of CAPTUREASSERT / CAPTURECLEAR matched:
   - If `OFFSETASSERT/CLEAR` enabled: `pps_add_offset(&ts_real, &params.{assert,clear}_off_tu)`.
   - Save `{assert,clear}_tu = ts_real`; bump `{assert,clear}_sequence`.
7. `pps_kc_event(pps, ts, event)` (in-kernel disciplining).
8. If anything captured: bump `last_ev`, `wake_up_interruptible_all(&pps->queue)`, `kill_fasync(POLL_IN)`.
9. Release lock.

Ioctl `pps_cdev_ioctl(file, cmd, arg)`:
- `PPS_GETPARAMS` — copy `pps->params` under lock to user (clears `flags` first to avoid leak).
- `PPS_SETPARAMS` — CAP_SYS_TIME; validate mode bits; store params; restore read-only fields; set api_version; zero flags.
- `PPS_GETCAP` — `put_user(info.mode)`.
- `PPS_FETCH` — `pps_cdev_pps_fetch(pps, fdata)` blocks; on return, copy `{assert,clear}_sequence/_tu`, `current_mode` to user; update `last_fetched_ev`.
- `PPS_KC_BIND` — CAP_SYS_TIME; validate consumer/format/edge; dispatch `pps_kc_bind`.
- default → -ENOTTY.

Compat ioctl `pps_cdev_compat_ioctl`:
- Rewrite cmd via `_IOC` macros (size differs because `time_t` differs).
- For PPS_FETCH only: explicit `pps_fdata_compat` ↔ `pps_fdata` conversion.
- Other cmds: fall through to native ioctl.

Poll `pps_cdev_poll`: `poll_wait(file, &pps->queue, wait)`; return `EPOLLIN|EPOLLRDNORM` iff `last_fetched_ev != last_ev`.

Sysfs assert/clear show: format `"%lld.%09d#%d"` (sec, nsec, sequence) iff `info.mode` has corresponding CAPTURE bit.

Subsystem init `pps_init` (subsys_initcall):
1. `class_register(&pps_class)` (groups = pps_groups).
2. `__register_chrdev(0, 0, PPS_MAX_SOURCES, "pps", &pps_cdev_fops)` → pps_major.

## Hardening

(Inherits row-1 features from `drivers/char/00-overview.md` § Hardening.)

pps-core specific reinforcement:

- **`PPS_SETPARAMS` zeroes `assert_off_tu.flags`/`clear_off_tu.flags`** explicitly — defeats info-leak of stack/userspace bits via `PPS_GETPARAMS`.
- **CAP_SYS_TIME** required for both `PPS_SETPARAMS` and `PPS_KC_BIND` — non-root cannot reconfigure or steal the disciplining bind.
- **`PPS_KC_BIND` accepts exactly one consumer (PPS_KC_HARDPPS) + one format (TSPEC) + edge in CAPTUREBOTH** — refuses arbitrary kernel-side handler attach.
- **idr range bounded to `PPS_MAX_SOURCES` (=128)** — defeats source-flood DoS.
- **`pps->lock` is IRQ-safe** — `pps_event` runs in IRQ context; ioctl + sysfs paths take the same lock with `spin_lock_irq` / `irqsave`.
- **`pps_unregister_source` clears `lookup_cookie` to NULL before cdev unregister** — defeats stale cookie reuse after teardown.
- **`get_device` taken in `pps_register_cdev` and matched in `pps_unregister_cdev`** — refcount keeps `pps_device` alive across all in-flight fds.
- **`pps_cdev_open` validates idr presence + takes per-device get_device** — defeats minor-rebinding race.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab cache for `pps_device`; all userland copies via `copy_to_user`/`copy_from_user` strictly bounded by `sizeof(pps_kparams)`, `sizeof(pps_fdata)`, `sizeof(pps_fdata_compat)`, `sizeof(pps_bind_args)`; no caller-provided length.
- **PAX_KERNEXEC** — PPS framework core in W^X kernel text; `pps_cdev_fops`, `pps_class`, sysfs attribute group, and per-source `info.echo` callback dispatch live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `pps_cdev_ioctl`, `pps_cdev_compat_ioctl`, `pps_event` (IRQ), `pps_cdev_pps_fetch`, `pps_register_source`.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-source `dev` refcount; `get_device`/`put_device` paired across `pps_idr_get`, `pps_cdev_open`, `pps_cdev_release`, `pps_register_cdev`, `pps_unregister_cdev`; overflow trap defeats open/unregister race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `pps_device` (so stale `assert_tu`/`clear_tu`/`info.echo` callback pointer cannot bleed across idr-slot reuse).
- **PAX_UDEREF** — SMAP/PAN enforced on every ioctl userland entry; explicit `copy_from_user`/`copy_to_user` with `-EFAULT` propagation on every branch.
- **PAX_RAP / kCFI** — `pps_cdev_fops` (`poll`/`fasync`/`unlocked_ioctl`/`compat_ioctl`/`open`/`release`), per-source `info.echo` callback, and `pps_kc_*` dispatch table marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-source `lookup_cookie` pointer disclosure behind CAP_SYSLOG; suppress `%p` in `dev_dbg` traces.
- **GRKERNSEC_DMESG** — restrict source-flood, unsupported-mode, and unspecified-format banners to CAP_SYSLOG so attackers cannot probe PPS source layout via dmesg.
- **`/dev/pps*` CAP_SYS_TIME-equivalent for write** — `PPS_SETPARAMS` and `PPS_KC_BIND` already gated on CAP_SYS_TIME; read-only ioctls (`PPS_GETPARAMS`/`PPS_GETCAP`/`PPS_FETCH`) accessible per node-permission policy; recommended `/dev/pps*` mode `0640 root:dialout` so non-CAP_SYS_TIME readers still need group membership.
- **Ioctl PAX_USERCOPY** — every `copy_from_user`/`copy_to_user` uses an exact `sizeof(struct)` constant from `<uapi/linux/pps.h>`; whitelist enforces this size in the PAX_USERCOPY slab descriptor for `pps_device`.
- **Time-disciplining stat sanitize** — `pps->params.assert_off_tu.flags` and `clear_off_tu.flags` zeroed inside `PPS_SETPARAMS` under `pps->lock` before unlock, ensuring `PPS_GETPARAMS` cannot leak uninitialized userspace flag bits.
- **Source-name allowlist** — `info.name` and `info.path` are kernel-driver-supplied (registered by `pps_register_source` callers — `pps-gpio`, `pps-ldisc`, `pps-parport`, `pps-phc`); framework copies them via `pps->info = *info`, never via user-controllable string; sysfs `name`/`path` show prints with `%s` and the source is in `__ro_after_init` driver data.
- **`PPS_KC_BIND` consumer allowlist** — `bind_args.consumer == PPS_KC_HARDPPS` is the only accepted value; framework refuses generic kernel-consumer attach, eliminating an arbitrary-callback-into-pps_event vector.
- **Edge allowlist** — `bind_args.edge & ~PPS_CAPTUREBOTH` must be zero; both edges or single-edge from the {CAPTUREASSERT, CAPTURECLEAR} set; defeats out-of-range edge bits.
- **Per-source IRQ-safe lock** — `pps->lock` is taken with `spin_lock_irqsave` in `pps_event` and `spin_lock_irq` in `PPS_SETPARAMS`/`PPS_FETCH`; framework guarantees no sleeping callout under the lock.
- **CAP_SYSLOG for dev_dbg/_err leaks** — when CONFIG_DYNAMIC_DEBUG=y enables `dev_dbg`, the resulting messages join the dmesg ring subject to GRKERNSEC_DMESG policy.

Rationale: PPS sources drive the host clock-disciplining loop via `hardpps`; an attacker who can either (a) trigger arbitrary `pps_event` calls or (b) hijack `PPS_KC_BIND` can skew NTP-disciplined clocks, fingerprint timing sources, or wedge the disciplining loop. RAP/kCFI on `info.echo` and `pps_kc_*` dispatch, CAP_SYS_TIME gates on every write ioctl, idr range cap to `PPS_MAX_SOURCES`, and explicit `flags`-zeroing on PARAMS-write convert the framework from "trust source driver" into a structural enforcement boundary around the kernel disciplining surface.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `idr_bound` | OOB | `pps_register_cdev` returns -EBUSY iff the system already has `PPS_MAX_SOURCES` registered; no minor outside `[0, PPS_MAX_SOURCES)`. |
| `setparams_subset` | INVARIANT | `PPS_SETPARAMS` rejects every `mode` with bits outside `info.mode`. |
| `kc_bind_allowlist` | INVARIANT | `PPS_KC_BIND` rejects every consumer != `PPS_KC_HARDPPS`, every format != `PPS_TSFMT_TSPEC`, every edge with bits outside `PPS_CAPTUREBOTH`. |
| `params_no_leak` | INFO-FLOW | every `PPS_GETPARAMS` result has `assert_off_tu.flags == clear_off_tu.flags == 0` if no `PPS_OFFSET*` was set by `PPS_SETPARAMS`. |

### Layer 2: TLA+

`models/pps/event_fetch.tla` (parent-declared): proves `pps_event` (IRQ context) and `PPS_FETCH` (process context) interleave correctly under `pps->lock`; `last_ev`/`last_fetched_ev` monotonicity guarantees fetcher observes every event at-most-once.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `pps_event` post: every successful capture bumps `last_ev` exactly once; wakeups fired iff bumped | `PpsDevice::event` |
| `PPS_FETCH` post: returned `assert/clear_sequence` matches `pps->{assert,clear}_sequence` at moment of unlock; `last_fetched_ev` equals `last_ev` at unlock | `PpsDevice::cdev_pps_fetch` |
| `pps_unregister_source` post: `lookup_cookie == NULL` and idr no longer maps `id` to the freed device | `PpsDevice::unregister_source` |

### Layer 4: Verus/Creusot functional

Event-to-userspace round-trip: source IRQ → `pps_event` → wake_up → `PPS_FETCH` returns the captured `pps_ktime` with `sequence == bumped_value`; encoded as Verus invariant chained with source-driver IRQ postcondition.

## Out of Scope

- Per-source client drivers (`pps-gpio`, `pps-ldisc`, `pps-parport`, `pps-phc`) — separate future Tier-3
- PPS generator framework (`drivers/pps/generators/`) — separate future Tier-3
- `pps_kc_event` hardpps disciplining internals (`pps/kc.c`) — separate future Tier-3
- 32-bit-only userland paths (covered via compat_ioctl above)
- Implementation code
