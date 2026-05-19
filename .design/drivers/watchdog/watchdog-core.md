# Tier-3: drivers/watchdog/watchdog_core.c — Watchdog core (register/unregister, chardev, keepalive)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/watchdog/00-overview.md
upstream-paths:
  - drivers/watchdog/watchdog_core.c (~495 lines)
  - drivers/watchdog/watchdog_core.h
  - drivers/watchdog/watchdog_dev.c (cdev / keepalive / fops; cross-referenced)
  - drivers/watchdog/watchdog_pretimeout.c (governor lookup; cross-referenced)
  - include/linux/watchdog.h
-->

## Summary

The watchdog core is the in-kernel framework that drivers register against to expose a `/dev/watchdogN` character device and a uniform set of ioctls (`WDIOC_*`) to user space. Each device is a `struct watchdog_device` with a `const struct watchdog_ops *ops` table (`start`, `stop`, `ping`, `status`, `set_timeout`, `set_pretimeout`, `get_timeleft`, `restart`, `ioctl`), an info pointer (`struct watchdog_info`), a per-device IDA-allocated id, and a `struct watchdog_core_data *wd_data` opaque core handle. The core multiplexes three lifecycles: (a) **deferred registration** — drivers may call `watchdog_register_device()` before the chardev infrastructure is ready (`subsys_initcall_sync`), in which case the device is parked on `wtd_deferred_reg_list` and registered later; (b) **hardware keepalive** — when userspace is not actively pinging (or has not opened the device yet) but the watchdog HW is running (`WDOG_HW_RUNNING`), a `kthread_worker` (in `watchdog_dev.c`) is driven by an `hrtimer` to call `ops->ping()` (or `ops->start()`) before each `max_hw_heartbeat_ms` deadline; (c) **pretimeout governor** — if the watchdog supports a pretimeout interrupt (`WDIOF_PRETIMEOUT` or `CONFIG_WATCHDOG_HRTIMER_PRETIMEOUT`), the driver calls `watchdog_notify_pretimeout(wdd)` which dispatches to a `struct watchdog_governor` (noop / panic / userspace) installed via `watchdog_register_governor()`. Critical for: bare-metal liveness, container/embedded-system runtime watchdogs, kdump-on-stall, and panic-on-pretimeout policy.

This Tier-3 covers `drivers/watchdog/watchdog_core.c` (~495 lines), the `struct watchdog_core_data` opaque from `drivers/watchdog/watchdog_core.h`, and the cdev / keepalive surface from `drivers/watchdog/watchdog_dev.c` insofar as it is observable through the core's contracts.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct watchdog_device` | per-device descriptor (info, ops, timeouts, status, id) | `WatchdogDevice` |
| `struct watchdog_ops` | per-driver vtable | `WatchdogOps` (trait) |
| `struct watchdog_info` | UAPI identity / firmware version / options | shared UAPI |
| `struct watchdog_core_data` | core-private: cdev, mutex, hrtimer, kthread_work, deadlines | `WatchdogCoreData` |
| `watchdog_register_device()` | per-driver registration | `WatchdogCore::register_device` |
| `watchdog_unregister_device()` | per-driver teardown | `WatchdogCore::unregister_device` |
| `devm_watchdog_register_device()` | per-devres managed register | `WatchdogCore::devm_register_device` |
| `watchdog_init_timeout()` | per-device timeout init (param > DT > default) | `WatchdogCore::init_timeout` |
| `watchdog_set_restart_priority()` | per-device restart-handler priority | `WatchdogCore::set_restart_priority` |
| `watchdog_set_drvdata()` / `watchdog_get_drvdata()` | per-driver private blob | inline helpers |
| `watchdog_active()` / `watchdog_hw_running()` | per-status query | inline helpers |
| `watchdog_check_min_max_timeout()` | per-device sanity | `WatchdogCore::check_min_max_timeout` |
| `___watchdog_register_device()` | inner registration (ida_alloc, dev_register, notifiers) | `WatchdogCore::register_device_inner` |
| `watchdog_deferred_registration()` | per-subsys_initcall_sync drain | `WatchdogCore::drain_deferred` |
| `watchdog_reboot_notifier()` | per-`WDOG_STOP_ON_REBOOT` shutdown hook | `WatchdogCore::reboot_notifier` |
| `watchdog_restart_notifier()` | per-system-restart hook | `WatchdogCore::restart_notifier` |
| `watchdog_pm_notifier()` | per-`WDOG_NO_PING_ON_SUSPEND` PM hook | `WatchdogCore::pm_notifier` |
| `watchdog_dev_register()` / `_unregister()` | per-cdev attach (defined in watchdog_dev.c) | `WatchdogDev::register` / `unregister` |
| `watchdog_ping()` / `__watchdog_ping()` | per-keepalive | `WatchdogDev::ping` / `ping_inner` |
| `watchdog_ping_work()` | per-kthread\_work HW-keepalive callback | `WatchdogDev::ping_work` |
| `watchdog_timer_expired()` | per-hrtimer next-keepalive trigger | `WatchdogDev::timer_expired` |
| `watchdog_worker_should_ping()` | per-keepalive predicate | `WatchdogDev::worker_should_ping` |
| `watchdog_need_worker()` | per-decide-sw-keepalive | `WatchdogDev::need_worker` |
| `watchdog_open()` / `_release()` | per-cdev open/close (single-open) | `WatchdogDev::open` / `release` |
| `watchdog_write()` | per-cdev write — keepalive + magic 'V' | `WatchdogDev::write` |
| `watchdog_ioctl()` | per-WDIOC_* dispatch | `WatchdogDev::ioctl` |
| `watchdog_notify_pretimeout()` | per-pretimeout governor dispatch | `WatchdogPretimeout::notify` |
| `WDOG_ACTIVE` / `WDOG_NO_WAY_OUT` / `WDOG_STOP_ON_REBOOT` / `WDOG_HW_RUNNING` / `WDOG_STOP_ON_UNREGISTER` / `WDOG_NO_PING_ON_SUSPEND` | status bits (`wdd->status`) | `WatchdogStatus` bitset |
| `_WDOG_DEV_OPEN` / `_WDOG_ALLOW_RELEASE` / `_WDOG_KEEPALIVE` | internal `wd_data->status` bits | `WatchdogInternalStatus` bitset |
| `MAX_DOGS` (= 32) | per-ida upper bound | const `MAX_DOGS = 32` |
| `WATCHDOG_NOWAYOUT` / `WATCHDOG_NOWAYOUT_INIT_STATUS` | per-`CONFIG_WATCHDOG_NOWAYOUT` policy | build-config const |
| `stop_on_reboot` (module param) | per-module-load reboot policy override | param `stop_on_reboot: Option<bool>` |
| `watchdog_ida` (`DEFINE_IDA`) | per-id allocator across all watchdogs | `WatchdogCore::ida` |

## Compatibility contract

REQ-1: `struct watchdog_device` shape (UAPI-shaped consumer-facing):
- id: i32 — kernel-allocated (IDA, range `[0, MAX_DOGS)`).
- parent: `*Device` — bus parent (may be NULL for legacy / soft-watchdog).
- groups: `**AttributeGroup` — sysfs attribute groups (optional).
- info: `*const WatchdogInfo` — identity (UAPI: `options`, `firmware_version`, `identity[32]`).
- ops: `*const WatchdogOps` — driver vtable (mandatory).
- gov: `*const WatchdogGovernor` — pretimeout governor (may be NULL).
- bootstatus: u32 — boot-time `WDIOF_*` from HW.
- timeout: u32 — current timeout (seconds).
- pretimeout: u32 — pre-timeout (seconds).
- min_timeout / max_timeout: u32 — bounds in seconds (zero = unused).
- min_hw_heartbeat_ms / max_hw_heartbeat_ms: u32 — HW heartbeat bounds in milliseconds; `max_hw_heartbeat_ms` non-zero ⟹ replaces `max_timeout`.
- reboot_nb / restart_nb / pm_nb: per-device `notifier_block`s.
- driver_data: `*mut ()` — accessed only via `watchdog_{set,get}_drvdata`.
- wd_data: `*mut WatchdogCoreData` — opaque to driver.
- status: bitset of `WDOG_ACTIVE | WDOG_NO_WAY_OUT | WDOG_STOP_ON_REBOOT | WDOG_HW_RUNNING | WDOG_STOP_ON_UNREGISTER | WDOG_NO_PING_ON_SUSPEND`.
- deferred: list node for `wtd_deferred_reg_list`.

REQ-2: `struct watchdog_ops` vtable:
- owner: `*Module` — module ownership (for `try_module_get` on `open`).
- start(wdd) -> i32 — mandatory; arm the HW watchdog.
- stop(wdd) -> i32 — optional; absent ⟹ HW cannot be stopped (sets `WDOG_HW_RUNNING` on every "stop"). Required if `max_hw_heartbeat_ms == 0`.
- ping(wdd) -> i32 — optional; absent ⟹ `start()` is reused for ping.
- status(wdd) -> u32 — optional; report HW `WDIOF_*` state.
- set_timeout(wdd, t) -> i32 — optional; honors `WDIOF_SETTIMEOUT`.
- set_pretimeout(wdd, t) -> i32 — optional.
- get_timeleft(wdd) -> u32 — optional.
- restart(wdd, action, data) -> i32 — optional; registers as a kernel restart handler.
- ioctl(wdd, cmd, arg) -> i64 — optional escape hatch; return `-ENOIOCTLCMD` for fallthrough.

REQ-3: `WatchdogCore::register_device(wdd)`:
- /* Outer gate by deferred-registration mutex */
- mutex_lock(&wtd_deferred_reg_mutex).
- if !wtd_deferred_reg_done:
  - list_add_tail(&wdd.deferred, &wtd_deferred_reg_list).
  - mutex_unlock. return 0.
- ret = `__watchdog_register_device(wdd)`.
- mutex_unlock. return ret.

REQ-4: `WatchdogCore::register_device_inner(wdd)` (`___watchdog_register_device`):
- /* Validate */
- if !wdd ∨ !wdd.info ∨ !wdd.ops: return -EINVAL.
- if !wdd.ops.start ∨ (!wdd.ops.stop ∧ wdd.max_hw_heartbeat_ms == 0): return -EINVAL.
- `watchdog_check_min_max_timeout(wdd)`.
- /* IDA-allocate id, preferring DT alias */
- id = -1.
- if wdd.parent ∧ of_alias_get_id(wdd.parent.of_node, "watchdog") >= 0:
  - id = ida_alloc_range(&watchdog_ida, alias, alias, GFP_KERNEL).
- if id < 0: id = ida_alloc_max(&watchdog_ida, MAX_DOGS - 1, GFP_KERNEL).
- if id < 0: return id.
- wdd.id = id.
- /* Attach cdev (legacy + per-watchdogN) */
- ret = watchdog_dev_register(wdd).
- if ret:
  - ida_free(&watchdog_ida, id).
  - /* id == 0 ∧ ret == -EBUSY ⟹ legacy /dev/watchdog already owned by old module; retry id >= 1 */
  - if !(id == 0 ∧ ret == -EBUSY): return ret.
  - id = ida_alloc_range(&watchdog_ida, 1, MAX_DOGS - 1, GFP_KERNEL).
  - if id < 0: return id.
  - wdd.id = id.
  - ret = watchdog_dev_register(wdd).
  - if ret: ida_free; return ret.
- /* Module param override */
- if stop_on_reboot != -1:
  - if stop_on_reboot: set_bit(WDOG_STOP_ON_REBOOT, &wdd.status).
  - else: clear_bit(WDOG_STOP_ON_REBOOT, &wdd.status).
- /* Reboot notifier */
- if test_bit(WDOG_STOP_ON_REBOOT, &wdd.status):
  - if !wdd.ops.stop: warn "stop_on_reboot not supported".
  - else:
    - wdd.reboot_nb.notifier_call = watchdog_reboot_notifier.
    - ret = register_reboot_notifier(&wdd.reboot_nb).
    - if ret: unregister cdev + ida_free + return ret.
- /* Restart handler */
- if wdd.ops.restart:
  - wdd.restart_nb.notifier_call = watchdog_restart_notifier.
  - ret = register_restart_handler(&wdd.restart_nb).
  - if ret: warn (non-fatal).
- /* PM notifier */
- if test_bit(WDOG_NO_PING_ON_SUSPEND, &wdd.status):
  - wdd.pm_nb.notifier_call = watchdog_pm_notifier.
  - ret = register_pm_notifier(&wdd.pm_nb).
  - if ret: warn (non-fatal).
- return 0.

REQ-5: `WatchdogCore::unregister_device(wdd)`:
- mutex_lock(&wtd_deferred_reg_mutex).
- if wtd_deferred_reg_done:
  - if wdd.ops.restart: unregister_restart_handler(&wdd.restart_nb).
  - if test_bit(WDOG_STOP_ON_REBOOT, &wdd.status): unregister_reboot_notifier(&wdd.reboot_nb).
  - watchdog_dev_unregister(wdd).
  - ida_free(&watchdog_ida, wdd.id).
- else:
  - list_del(&wdd.deferred) from wtd_deferred_reg_list.
- mutex_unlock.

REQ-6: `WatchdogCore::devm_register_device(dev, wdd)`:
- rcwdd = devres_alloc(devm_watchdog_unregister_device, sizeof(*rcwdd), GFP_KERNEL).
- if !rcwdd: return -ENOMEM.
- ret = `WatchdogCore::register_device(wdd)`.
- if !ret: *rcwdd = wdd; devres_add(dev, rcwdd).
- else: devres_free(rcwdd).
- return ret.
- /* On dev detach: `devm_watchdog_unregister_device` calls `watchdog_unregister_device(*rcwdd)`. */

REQ-7: `WatchdogCore::init_timeout(wdd, timeout_parm, dev)`:
- /* Module param takes precedence over DT, both over default */
- `watchdog_check_min_max_timeout(wdd)`.
- if timeout_parm != 0:
  - if !watchdog_timeout_invalid(wdd, timeout_parm): wdd.timeout = timeout_parm; return 0.
  - pr_err "driver supplied timeout out of range"; ret = -EINVAL.
- if dev ∧ device_property_read_u32(dev, "timeout-sec", &t) == 0:
  - if t != 0 ∧ !watchdog_timeout_invalid(wdd, t): wdd.timeout = t; return 0.
  - pr_err "DT supplied timeout out of range"; ret = -EINVAL.
- /* Fall back to default; warn if any value was invalid */
- if ret < 0 ∧ wdd.timeout != 0: pr_warn "falling back to default".
- return ret.

REQ-8: `WatchdogCore::check_min_max_timeout(wdd)`:
- if wdd.max_hw_heartbeat_ms == 0 ∧ wdd.min_timeout > wdd.max_timeout:
  - pr_info "Invalid min and max timeout values, resetting to 0!".
  - wdd.min_timeout = 0; wdd.max_timeout = 0.

REQ-9: `WatchdogCore::reboot_notifier(nb, code, data)`:
- wdd = container_of(nb, struct watchdog_device, reboot_nb).
- if code ∈ {SYS_DOWN, SYS_HALT, SYS_POWER_OFF}:
  - if watchdog_hw_running(wdd):
    - ret = wdd.ops.stop(wdd).
    - trace_watchdog_stop(wdd, ret).
    - if ret != 0: return NOTIFY_BAD.
- return NOTIFY_DONE.

REQ-10: `WatchdogCore::restart_notifier(nb, action, data)`:
- wdd = container_of(nb, struct watchdog_device, restart_nb).
- ret = wdd.ops.restart(wdd, action, data).
- if ret != 0: return NOTIFY_BAD.
- return NOTIFY_DONE.

REQ-11: `WatchdogCore::pm_notifier(nb, mode, data)`:
- wdd = container_of(nb, struct watchdog_device, pm_nb).
- match mode:
  - PM_HIBERNATION_PREPARE | PM_RESTORE_PREPARE | PM_SUSPEND_PREPARE ⟹ ret = watchdog_dev_suspend(wdd).
  - PM_POST_HIBERNATION | PM_POST_RESTORE | PM_POST_SUSPEND ⟹ ret = watchdog_dev_resume(wdd).
- return ret != 0 ? NOTIFY_BAD : NOTIFY_DONE.

REQ-12: `WatchdogCore::set_restart_priority(wdd, priority)`:
- wdd.restart_nb.priority = priority.
- /* Priority guideline: 0 = last-resort, 128 = default, 255 = preempt-all. */

REQ-13: `WatchdogCore::drain_deferred()` (`watchdog_deferred_registration`):
- mutex_lock(&wtd_deferred_reg_mutex).
- wtd_deferred_reg_done = true.
- while !list_empty(&wtd_deferred_reg_list):
  - wdd = list_first_entry(&wtd_deferred_reg_list, struct watchdog_device, deferred).
  - list_del(&wdd.deferred).
  - `__watchdog_register_device(wdd)`.
- mutex_unlock.

REQ-14: Init/exit ordering (`subsys_initcall_sync`):
- watchdog_init: watchdog_dev_init() (allocates `dev_t`, creates `watchdog_kworker`, sets up `watchdog_class`); then drain_deferred.
- watchdog_exit: watchdog_dev_exit(); ida_destroy(&watchdog_ida).

REQ-15: `WatchdogCoreData` opaque (per `wd_data`):
- dev: embedded `struct device` (for sysfs / cdev parent).
- cdev: embedded `struct cdev` (the per-watchdogN char device).
- wdd: back-pointer to `watchdog_device` (NULL after `_unregister`).
- lock: `mutex` — serializes all wd_data state transitions (ping/start/stop/ioctl/write).
- last_keepalive: ktime_t — last user-driven ping.
- last_hw_keepalive: ktime_t — last `ops->ping`/`ops->start` invocation.
- open_deadline: ktime_t — until-when the soft worker keeps pinging an HW-running watchdog with no userspace open (init from `CONFIG_WATCHDOG_OPEN_TIMEOUT`; KTIME_MAX once opened).
- timer: `hrtimer` (`HRTIMER_MODE_REL_HARD`) — schedules the next keepalive.
- work: `kthread_work` — runs `watchdog_ping_work()` on the global `watchdog_kworker`.
- pretimeout_timer: hrtimer (only `CONFIG_WATCHDOG_HRTIMER_PRETIMEOUT`).
- status: bitset of `_WDOG_DEV_OPEN | _WDOG_ALLOW_RELEASE | _WDOG_KEEPALIVE`.

REQ-16: HW-keepalive (`__watchdog_ping(wdd)`):
- earliest_keepalive = wd_data.last_hw_keepalive + ms_to_ktime(wdd.min_hw_heartbeat_ms).
- now = ktime_get().
- if ktime_after(earliest_keepalive, now):
  - /* Too early: re-arm hrtimer */
  - hrtimer_start(&wd_data.timer, ktime_sub(earliest_keepalive, now), HRTIMER_MODE_REL_HARD).
  - return 0.
- wd_data.last_hw_keepalive = now.
- if wdd.ops.ping: err = wdd.ops.ping(wdd); trace_watchdog_ping(wdd, err).
- else:           err = wdd.ops.start(wdd); trace_watchdog_start(wdd, err).
- if err == 0: watchdog_hrtimer_pretimeout_start(wdd).
- watchdog_update_worker(wdd).
- return err.

REQ-17: User-keepalive (`watchdog_ping(wdd)`):
- if !watchdog_hw_running(wdd): return 0.
- set_bit(_WDOG_KEEPALIVE, &wd_data.status).
- wd_data.last_keepalive = ktime_get().
- return `__watchdog_ping(wdd)`.

REQ-18: `WatchdogDev::need_worker(wdd)`:
- hm = wdd.max_hw_heartbeat_ms.
- t  = wdd.timeout * 1000.
- return (hm != 0 ∧ watchdog_active(wdd) ∧ t > hm) ∨ (t != 0 ∧ !watchdog_active(wdd) ∧ watchdog_hw_running(wdd)).
- /* Two cases: (a) user-requested timeout exceeds HW max → SW must extend; (b) HW running w/o user opening → SW must feed until open_deadline. */

REQ-19: `WatchdogDev::worker_should_ping(wd_data)`:
- wdd = wd_data.wdd.
- if !wdd: return false.
- if watchdog_active(wdd): return true.
- return watchdog_hw_running(wdd) ∧ !watchdog_past_open_deadline(wd_data).

REQ-20: `WatchdogDev::ping_work(work)` (the `kthread_work` callback):
- wd_data = container_of(work, struct watchdog_core_data, work).
- mutex_lock(&wd_data.lock).
- if worker_should_ping(wd_data): `__watchdog_ping(wd_data.wdd)`.
- mutex_unlock.

REQ-21: `WatchdogDev::timer_expired(timer)` (hrtimer callback):
- wd_data = container_of(timer, struct watchdog_core_data, timer).
- kthread_queue_work(watchdog_kworker, &wd_data.work).
- return HRTIMER_NORESTART.
- /* Re-arm is handled by `__watchdog_ping` via `watchdog_update_worker`. */

REQ-22: `WatchdogDev::open(inode, file)`:
- /* Legacy /dev/watchdog: imajor == MISC_MAJOR ⟹ wd_data = old_wd_data; else container_of(inode->i_cdev). */
- if test_and_set_bit(_WDOG_DEV_OPEN, &wd_data.status): return -EBUSY.   /* single-open */
- hw_running = watchdog_hw_running(wdd).
- if !hw_running ∧ !try_module_get(wdd.ops.owner): goto out_clear; return -EBUSY.
- err = watchdog_start(wdd).
- if err < 0: goto out_mod.
- file.private_data = wd_data.
- if !hw_running: get_device(&wd_data.dev).
- wd_data.open_deadline = KTIME_MAX.
- return stream_open(inode, file).

REQ-23: `WatchdogDev::release(inode, file)`:
- mutex_lock(&wd_data.lock).
- /* Stop iff magic 'V' received and !WDOG_NO_WAY_OUT */
- if !watchdog_active(wdd): err = 0.
- else if test_and_clear_bit(_WDOG_ALLOW_RELEASE, &wd_data.status) ∨ !(wdd.info.options & WDIOF_MAGICCLOSE):
  - err = watchdog_stop(wdd).
- if err < 0: pr_crit "watchdog%d: did not stop"; watchdog_ping(wdd).
- watchdog_update_worker(wdd).
- clear_bit(_WDOG_DEV_OPEN, &wd_data.status).
- running = wdd ∧ watchdog_hw_running(wdd).
- mutex_unlock.
- if !running: module_put(wd_data.cdev.owner); put_device(&wd_data.dev).
- return 0.

REQ-24: `WatchdogDev::write(file, data, len, ppos)`:
- if len == 0: return 0.
- clear_bit(_WDOG_ALLOW_RELEASE, &wd_data.status).
- /* Scan for magic 'V' */
- for i in 0..len:
  - if get_user(c, data + i): return -EFAULT.
  - if c == 'V': set_bit(_WDOG_ALLOW_RELEASE, &wd_data.status).
- /* Any write is a keepalive */
- mutex_lock(&wd_data.lock); err = wdd ? watchdog_ping(wdd) : -ENODEV; mutex_unlock.
- return err < 0 ? err : len.

REQ-25: `WatchdogDev::ioctl(file, cmd, arg)` (under wd_data.lock):
- Dispatch via wdd.ops.ioctl first; if -ENOIOCTLCMD then handle:
  - WDIOC_GETSUPPORT     ⟹ copy_to_user(argp, wdd.info, sizeof(struct watchdog_info)).
  - WDIOC_GETSTATUS      ⟹ put_user(watchdog_get_status(wdd), p).
  - WDIOC_GETBOOTSTATUS  ⟹ put_user(wdd.bootstatus, p).
  - WDIOC_SETOPTIONS(val):
    - WDIOS_DISABLECARD ⟹ watchdog_stop(wdd).
    - WDIOS_ENABLECARD  ⟹ watchdog_start(wdd).
  - WDIOC_KEEPALIVE     ⟹ if !(wdd.info.options & WDIOF_KEEPALIVEPING) return -EOPNOTSUPP; else watchdog_ping(wdd).
  - WDIOC_SETTIMEOUT(val) ⟹ watchdog_set_timeout(wdd, val); on success ping; fallthrough.
  - WDIOC_GETTIMEOUT    ⟹ if wdd.timeout == 0: -EOPNOTSUPP; else put_user(wdd.timeout, p).
  - WDIOC_GETTIMELEFT   ⟹ put_user(get_timeleft, p).
  - WDIOC_SETPRETIMEOUT(val) ⟹ watchdog_set_pretimeout(wdd, val).
  - WDIOC_GETPRETIMEOUT ⟹ put_user(wdd.pretimeout, p).
  - default ⟹ -ENOTTY.

REQ-26: Pretimeout governor (`watchdog_notify_pretimeout(wdd)`):
- spin_lock(&pretimeout_lock).
- if wdd.gov: wdd.gov.pretimeout(wdd).
- spin_unlock.
- /* Governors: noop (default), panic, userspace (sysfs `pretimeout_governor`). */

REQ-27: Status bits (`wdd->status`):
- WDOG_ACTIVE (0): user has armed via WDIOC_SETOPTIONS(ENABLECARD) or open.
- WDOG_NO_WAY_OUT (1): nowayout; stop is refused with -EBUSY.
- WDOG_STOP_ON_REBOOT (2): stop on SYS_DOWN/HALT/POWER_OFF.
- WDOG_HW_RUNNING (3): HW is feeding regardless of user (e.g. boot-enabled).
- WDOG_STOP_ON_UNREGISTER (4): driver requests stop in `_unregister`.
- WDOG_NO_PING_ON_SUSPEND (5): SW worker frozen across suspend.

REQ-28: Internal bits (`wd_data->status`):
- _WDOG_DEV_OPEN (0): single-open lock.
- _WDOG_ALLOW_RELEASE (1): magic 'V' seen ⟹ next close stops.
- _WDOG_KEEPALIVE (2): a userspace keepalive happened (cleared on WDIOC_GETSTATUS).

REQ-29: Driver private data:
- watchdog_set_drvdata(wdd, data): wdd.driver_data = data (inline).
- watchdog_get_drvdata(wdd):       return wdd.driver_data (inline).
- /* No core lock; driver-internal use only; the field is treated as opaque by core. */

REQ-30: Tracepoints:
- `trace_watchdog_start(wdd, err)`, `trace_watchdog_stop(wdd, err)`, `trace_watchdog_ping(wdd, err)`, `trace_watchdog_set_timeout(wdd, t, err)`.

## Acceptance Criteria

- [ ] AC-1: `register_device` before `subsys_initcall_sync` ⟹ queued on `wtd_deferred_reg_list`; drained on init.
- [ ] AC-2: Missing `ops->start` or (missing `ops->stop` ∧ `max_hw_heartbeat_ms == 0`) ⟹ -EINVAL.
- [ ] AC-3: DT alias "watchdog" present ⟹ id matches alias; else first free in `[0, MAX_DOGS)`.
- [ ] AC-4: id 0 collision with legacy miscdev ⟹ retry in `[1, MAX_DOGS)`.
- [ ] AC-5: `stop_on_reboot=1` ⟹ reboot notifier registered; SYS_DOWN ⟹ ops->stop called.
- [ ] AC-6: `ops->restart` ⟹ restart handler registered at priority set by `set_restart_priority`.
- [ ] AC-7: `WDOG_NO_PING_ON_SUSPEND` ⟹ PM notifier suspends/resumes worker.
- [ ] AC-8: Userspace write of "V…" ⟹ next close stops the watchdog (unless nowayout).
- [ ] AC-9: WDIOC_KEEPALIVE without `WDIOF_KEEPALIVEPING` ⟹ -EOPNOTSUPP.
- [ ] AC-10: `max_hw_heartbeat_ms > 0` ∧ user `timeout` > hm ∧ active ⟹ kthread worker fires every `min(hm, next-deadline)` to feed HW.
- [ ] AC-11: Boot-enabled HW (WDOG_HW_RUNNING) with no user open ⟹ worker feeds until `open_deadline`.
- [ ] AC-12: Two opens of `/dev/watchdogN` ⟹ second returns -EBUSY (single-open).
- [ ] AC-13: `devm_watchdog_register_device` ⟹ unregister automatically on device detach.
- [ ] AC-14: Pretimeout governor "panic" + pretimeout fires ⟹ `panic()` invoked.
- [ ] AC-15: `watchdog_init_timeout`: module param valid ⟹ wins; else DT `timeout-sec` valid ⟹ wins; else default kept.

## Architecture

```
struct WatchdogDevice {
  id: i32,
  parent: Option<*Device>,
  groups: *const *const AttributeGroup,
  info: *const WatchdogInfo,
  ops: *const WatchdogOps,
  gov: Option<*const WatchdogGovernor>,
  bootstatus: u32,
  timeout: u32,
  pretimeout: u32,
  min_timeout: u32,
  max_timeout: u32,
  min_hw_heartbeat_ms: u32,
  max_hw_heartbeat_ms: u32,
  reboot_nb: NotifierBlock,
  restart_nb: NotifierBlock,
  pm_nb: NotifierBlock,
  driver_data: *mut (),
  wd_data: *mut WatchdogCoreData,
  status: AtomicU64,                 // WDOG_* bits
  deferred: ListHead,
}

struct WatchdogOps {
  owner: *Module,
  start:         fn(*WatchdogDevice) -> i32,                          // mandatory
  stop:          Option<fn(*WatchdogDevice) -> i32>,
  ping:          Option<fn(*WatchdogDevice) -> i32>,
  status:        Option<fn(*WatchdogDevice) -> u32>,
  set_timeout:   Option<fn(*WatchdogDevice, u32) -> i32>,
  set_pretimeout:Option<fn(*WatchdogDevice, u32) -> i32>,
  get_timeleft:  Option<fn(*WatchdogDevice) -> u32>,
  restart:       Option<fn(*WatchdogDevice, u64, *mut ()) -> i32>,
  ioctl:         Option<fn(*WatchdogDevice, u32, u64) -> i64>,
}

struct WatchdogCoreData {
  dev: Device,
  cdev: Cdev,
  wdd: Option<*WatchdogDevice>,
  lock: Mutex<()>,
  last_keepalive: KTime,
  last_hw_keepalive: KTime,
  open_deadline: KTime,
  timer: HrTimer,                    // HRTIMER_MODE_REL_HARD
  work: KthreadWork,                 // → watchdog_kworker
  pretimeout_timer: Option<HrTimer>, // cfg(watchdog_hrtimer_pretimeout)
  status: AtomicU64,                 // _WDOG_* bits
}
```

`WatchdogCore::register_device(wdd) -> Result<(), Errno>`:
1. /* Outer gate; queue if pre-init */
2. let g = wtd_deferred_reg_mutex.lock().
3. if !wtd_deferred_reg_done:
   - wtd_deferred_reg_list.push_back(&wdd.deferred).
   - return Ok(()).
4. WatchdogCore::register_device_inner(wdd).

`WatchdogCore::register_device_inner(wdd) -> Result<(), Errno>`:
1. /* Validate vtable */
2. if wdd.info.is_null() ∨ wdd.ops.is_null(): return Err(-EINVAL).
3. if wdd.ops.start.is_none() ∨ (wdd.ops.stop.is_none() ∧ wdd.max_hw_heartbeat_ms == 0): return Err(-EINVAL).
4. WatchdogCore::check_min_max_timeout(wdd).
5. /* Allocate id (DT-alias-first) */
6. let alias = wdd.parent.and_then(|p| of_alias_get_id(p.of_node, "watchdog")).
7. let id = match alias { Some(a) => ida_alloc_range(a, a)?, None => ida_alloc_max(MAX_DOGS-1)? }.
8. wdd.id = id.
9. /* Attach cdev */
10. WatchdogDev::register(wdd).or_else(|e| /* id==0 ∧ e==EBUSY ⟹ retry [1, MAX_DOGS) */ retry_with_id_gt_zero()).
11. /* stop_on_reboot module param override */
12. if let Some(v) = stop_on_reboot { wdd.status.bit(WDOG_STOP_ON_REBOOT).store(v) }.
13. /* Notifiers */
14. if wdd.status.bit(WDOG_STOP_ON_REBOOT).load() ∧ wdd.ops.stop.is_some():
    - wdd.reboot_nb.notifier_call = WatchdogCore::reboot_notifier.
    - register_reboot_notifier(&wdd.reboot_nb).map_err(rollback_cdev_ida)?.
15. if wdd.ops.restart.is_some():
    - wdd.restart_nb.notifier_call = WatchdogCore::restart_notifier.
    - register_restart_handler(&wdd.restart_nb).warn_on_err().
16. if wdd.status.bit(WDOG_NO_PING_ON_SUSPEND).load():
    - wdd.pm_nb.notifier_call = WatchdogCore::pm_notifier.
    - register_pm_notifier(&wdd.pm_nb).warn_on_err().
17. return Ok(()).

`WatchdogCore::unregister_device(wdd)`:
1. let g = wtd_deferred_reg_mutex.lock().
2. if wtd_deferred_reg_done:
   - if wdd.ops.restart.is_some(): unregister_restart_handler(&wdd.restart_nb).
   - if wdd.status.bit(WDOG_STOP_ON_REBOOT).load(): unregister_reboot_notifier(&wdd.reboot_nb).
   - WatchdogDev::unregister(wdd).
   - ida_free(wdd.id).
3. else:
   - wtd_deferred_reg_list.remove(&wdd.deferred).

`WatchdogCore::init_timeout(wdd, timeout_parm, dev) -> Result<(), Errno>`:
1. WatchdogCore::check_min_max_timeout(wdd).
2. let mut err = Ok(()).
3. if timeout_parm != 0:
   - if !watchdog_timeout_invalid(wdd, timeout_parm) { wdd.timeout = timeout_parm; return Ok(()); }
   - err = Err(-EINVAL).
4. if let Some(d) = dev { if let Ok(t) = d.property_read_u32("timeout-sec") {
   - if t != 0 ∧ !watchdog_timeout_invalid(wdd, t) { wdd.timeout = t; return Ok(()); }
   - err = Err(-EINVAL).
   } }
5. return err.

`WatchdogDev::ping_inner(wdd) -> Result<(), Errno>`:
1. let now = ktime_get().
2. let earliest = wd_data.last_hw_keepalive + ms_to_ktime(wdd.min_hw_heartbeat_ms).
3. if earliest > now:
   - hrtimer_start(&wd_data.timer, earliest - now, HRTIMER_MODE_REL_HARD).
   - return Ok(()).
4. wd_data.last_hw_keepalive = now.
5. let err = if let Some(p) = wdd.ops.ping { p(wdd) } else { wdd.ops.start(wdd) }.
6. if err == 0: watchdog_hrtimer_pretimeout_start(wdd).
7. WatchdogDev::update_worker(wdd).
8. err.

`WatchdogDev::ping_work(work)`:
1. let wd_data = container_of(work, WatchdogCoreData, work).
2. let _g = wd_data.lock.lock().
3. if WatchdogDev::worker_should_ping(wd_data): WatchdogDev::ping_inner(wd_data.wdd).

`WatchdogDev::timer_expired(timer) -> HrTimerRestart`:
1. let wd_data = container_of(timer, WatchdogCoreData, timer).
2. kthread_queue_work(&watchdog_kworker, &wd_data.work).
3. HrTimerRestart::NoRestart.

`WatchdogDev::open(inode, file) -> Result<i32, Errno>`:
1. /* Legacy /dev/watchdog vs /dev/watchdogN */
2. let wd_data = if inode.imajor() == MISC_MAJOR { old_wd_data } else { container_of(inode.i_cdev, WatchdogCoreData, cdev) }.
3. if wd_data.status.bit(_WDOG_DEV_OPEN).test_and_set(): return Err(-EBUSY).
4. let hw_running = watchdog_hw_running(wd_data.wdd).
5. if !hw_running ∧ !try_module_get(wdd.ops.owner) { wd_data.status.bit(_WDOG_DEV_OPEN).clear(); return Err(-EBUSY); }
6. watchdog_start(wd_data.wdd)?.
7. file.private_data = wd_data.
8. if !hw_running: get_device(&wd_data.dev).
9. wd_data.open_deadline = KTime::MAX.
10. stream_open(inode, file).

`WatchdogPretimeout::notify(wdd)`:
1. let _g = pretimeout_lock.lock().
2. if let Some(g) = wdd.gov: g.pretimeout(wdd).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ida_id_within_max_dogs` | INVARIANT | per-register: id ∈ [0, MAX_DOGS). |
| `legacy_id0_retry_drops_id0` | INVARIANT | per-EBUSY-on-id0: original id 0 freed before retry. |
| `ping_under_wd_data_lock` | INVARIANT | per-watchdog_ping: wd_data.lock held. |
| `single_open_excludes_second_open` | INVARIANT | per-_WDOG_DEV_OPEN: test_and_set returns true ⟹ -EBUSY. |
| `nowayout_blocks_stop` | INVARIANT | per-WDOG_NO_WAY_OUT: watchdog_stop returns -EBUSY. |
| `magic_v_required_for_close_stop` | INVARIANT | per-release: WDIOF_MAGICCLOSE ∧ !_WDOG_ALLOW_RELEASE ⟹ no stop. |
| `vtable_validated_pre_register` | INVARIANT | per-register: ops.start ∧ (ops.stop ∨ max_hw_heartbeat_ms != 0). |
| `min_hw_heartbeat_respected` | INVARIANT | per-ping_inner: now ≥ last_hw_keepalive + min_hw_heartbeat. |
| `deferred_drained_once` | INVARIANT | per-init: drain runs exactly once; wtd_deferred_reg_done sticky. |
| `register_rollback_balanced` | INVARIANT | per-error-path: cdev/ida/notifier rolled back on failure. |
| `timer_mode_rel_hard` | INVARIANT | per-hrtimer_start: mode == HRTIMER_MODE_REL_HARD. |
| `worker_only_when_needed` | INVARIANT | per-watchdog_update_worker: started ⟺ need_worker. |

### Layer 2: TLA+

`drivers/watchdog/watchdog-core.tla`:
- Per-register / unregister / open / release / write / ioctl / hw-keepalive worker / pretimeout dispatch.
- Properties:
  - `safety_single_open_excludes_concurrent_open` — per-cdev: at most one holder of _WDOG_DEV_OPEN.
  - `safety_nowayout_never_stops` — per-WDOG_NO_WAY_OUT: no transition stops the HW.
  - `safety_min_hw_heartbeat_observed` — per-ping_inner: HW ping cadence ≥ min_hw_heartbeat_ms.
  - `safety_max_hw_heartbeat_not_missed` — per-need_worker case (a): user-timeout > hm ∧ active ⟹ next ping ≤ hm.
  - `safety_boot_enabled_fed_until_deadline` — per-WDOG_HW_RUNNING ∧ !open ⟹ ping until open_deadline.
  - `safety_register_atomic_on_failure` — per-register: failure ⟹ no leaked id / notifier / cdev.
  - `liveness_register_eventually_drains` — per-deferred: subsys_initcall_sync ⟹ list drains.
  - `liveness_release_eventually_stops_or_pings` — per-release: terminates with stop or ping.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `register_device_inner` post: Ok ⟹ wdd.id allocated ∧ cdev attached ∧ notifiers (per flags) registered | `WatchdogCore::register_device_inner` |
| `register_device_inner` post: Err ⟹ no IDA leak ∧ no cdev leak ∧ no notifier leak | `WatchdogCore::register_device_inner` |
| `init_timeout` post: Ok ⟹ wdd.timeout is param ∨ DT (valid) | `WatchdogCore::init_timeout` |
| `ping_inner` post: returns 0 ⟹ ops.ping (or ops.start) was called this tick | `WatchdogDev::ping_inner` |
| `open` post: Ok ⟹ _WDOG_DEV_OPEN set ∧ module-ref held ∧ device-ref held (if !hw_running) | `WatchdogDev::open` |
| `release` post: !nowayout ∧ magic-V ⟹ watchdog stopped | `WatchdogDev::release` |
| `update_worker` post: hrtimer armed ⟺ need_worker | `WatchdogDev::update_worker` |
| `unregister_device` post: notifiers/cdev/ida all freed | `WatchdogCore::unregister_device` |

### Layer 4: Verus/Creusot functional

`Per-driver register → IDA-alloc id → cdev attach → notifier wiring → keepalive (user-write ∨ kthread_work via hrtimer) → governor on pretimeout → release with magic-V or nowayout → unregister teardown` semantic equivalence: per-Documentation/watchdog/watchdog-api.rst and Documentation/watchdog/watchdog-kernel-api.rst.

## Hardening

(Inherits row-1 features from `drivers/watchdog/00-overview.md` § Hardening.)

Watchdog-core reinforcement:

- **Per-vtable validation at register** — defense against per-incomplete-driver crash (no `start`, etc.).
- **Per-id bounded by MAX_DOGS=32** — defense against per-id-exhaustion attack via fake watchdog modules.
- **Per-legacy /dev/watchdog id-0 collision retry** — defense against per-legacy-module clobber.
- **Per-cdev single-open (`_WDOG_DEV_OPEN` test-and-set)** — defense against per-multi-pinger races.
- **Per-`WDOG_NO_WAY_OUT` rejects stop** — defense against per-userspace-disarm escalation.
- **Per-magic-'V' required for close-to-stop** — defense against per-buggy-process accidental disarm.
- **Per-min_hw_heartbeat_ms backoff** — defense against per-HW-violation (ping faster than HW allows).
- **Per-max_hw_heartbeat_ms SW extension** — defense against per-too-short-HW-timeout user-config.
- **Per-`watchdog_check_min_max_timeout` zero-on-bad-bounds** — defense against per-bogus driver bounds.
- **Per-mutex around all wd_data state** — defense against per-write/ioctl/release race.
- **Per-`HRTIMER_MODE_REL_HARD`** — defense against per-soft-IRQ-deferral missing the heartbeat under load.
- **Per-deferred-registration drain at `subsys_initcall_sync`** — defense against per-too-early-driver crash.
- **Per-devres `devm_watchdog_unregister_device`** — defense against per-leaked watchdog on probe-failure.
- **Per-PM notifier `WDOG_NO_PING_ON_SUSPEND`** — defense against per-suspend false-keepalive masking a hung HW.
- **Per-reboot notifier `WDOG_STOP_ON_REBOOT`** — defense against per-reboot watchdog firing mid-shutdown.

## Grsecurity/PaX-style Reinforcement

Beyond the upstream hardening above, Rookery layers the following grsec/PaX-style controls onto `drivers/watchdog/watchdog_*.c`:

- **PAX_USERCOPY** — bounds-checks `watchdog_info`, `pretimeout_governor` strings, and timeout integers in `ioctl(WDIOC_*)` so a malformed ioctl cannot overrun the kernel-side struct copy.
- **PAX_KERNEXEC** — keeps the `watchdog_ops` vtables (start/stop/ping/set_timeout/pretimeout) read-only after registration so a vendor-driver memory bug cannot rewrite the disarm path.
- **PAX_RANDKSTACK** — randomizes kernel-stack offset on each `write(/dev/watchdog)`/`ioctl` entry, defeating deterministic stack-spray against the long pretimeout-governor switch path.
- **PAX_REFCOUNT** — wraps `watchdog_device.id` and `wd_data.kref` so repeated open/close cycles cannot wrap the refcount and prematurely free the cdev.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `watchdog_core_data` and pretimeout-governor private state so residual driver pointers cannot be recovered from the freelist.
- **PAX_UDEREF** — enforces user/kernel pointer separation across `watchdog_write` (magic-'V' scan) and `watchdog_ioctl`.
- **PAX_RAP / kCFI** — forward-edge CFI on the `watchdog_ops` vtable plus the pretimeout-governor callback so a corrupted function pointer cannot redirect `wdd->ops->stop` to an attacker gadget.
- **GRKERNSEC_HIDESYM** — strips kernel pointers from watchdog probe/registration printks so unprivileged dmesg readers cannot leak driver vtable addresses.
- **GRKERNSEC_DMESG** — gates dmesg on `CAP_SYSLOG` so watchdog timeout/pretimeout panic traces do not leak addresses to unprivileged users.
- **CAP_SYS_ADMIN on `/dev/watchdog*`** — open is hard-gated; even with permissive udev an unprivileged process cannot disarm the system watchdog or program `min/max_hw_heartbeat_ms`.
- **Magic-'V' write protection enforced kernel-side** — `WDOG_NO_WAY_OUT` plus the magic-character scan in `watchdog_write` is treated as a security invariant, not a courtesy, so a buggy daemon cannot accidentally disarm the watchdog by writing arbitrary data.
- **Pretimeout governor switch under capability check** — selecting `pretimeout_governor` via sysfs is restricted to `CAP_SYS_ADMIN` to prevent unprivileged processes from steering panic/noop behavior away from the configured policy.
- **GRKERNSEC_HIDESYM on watchdog notifier traces** — reboot/PM notifier failure paths sanitize device pointers so a watchdog HW fault report does not leak kASLR-relative addresses.

Rationale: `/dev/watchdog` is the canonical liveness primitive for a hardened deployment — silent disarm or governor swap defeats the entire reset-on-hang guarantee, so capability gating and CFI on the ops vtable are layered on top of the upstream magic-'V' / `NO_WAY_OUT` model.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Individual driver implementations (e.g. `iTCO_wdt`, `softdog`, `sp5100_tco`) — covered per-driver in their own Tier-3s if expanded.
- Sysfs attribute group internals (`wdt_groups`) — covered alongside `drivers/base/core.c` Tier-3.
- Pretimeout governor implementations (noop/panic/userspace) — covered in `watchdog-pretimeout.md` Tier-3 if expanded.
- `CONFIG_WATCHDOG_HRTIMER_PRETIMEOUT` synthetic-pretimeout — covered in `watchdog_hrtimer_pretimeout.c` if expanded.
- UAPI `<linux/watchdog.h>` constants (`WDIOC_*`, `WDIOF_*`, `WDIOS_*`) — covered alongside `include/uapi/linux/watchdog.h` if expanded.
- Implementation code.
