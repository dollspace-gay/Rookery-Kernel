# Tier-3: drivers/rtc/class.c — RTC class subsystem

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/rtc/00-overview.md
upstream-paths:
  - drivers/rtc/class.c (~491 lines)
  - drivers/rtc/rtc-core.h
  - drivers/rtc/interface.c (rtc_initialize_alarm, rtc_aie_update_irq, rtc_uie_update_irq, rtc_pie_update_irq, rtc_timer_do_work)
  - drivers/rtc/dev.c (rtc_dev_init, rtc_dev_prepare, /dev/rtcN cdev + ioctl)
  - drivers/rtc/proc.c (rtc_proc_add_device, rtc_proc_del_device)
  - drivers/rtc/sysfs.c (rtc_get_dev_attribute_groups, date/time/alarm/wakealarm/offset/range sysfs)
  - include/linux/rtc.h
  - include/uapi/linux/rtc.h (RTC_AIE_ON / RTC_PIE_ON / RTC_UIE_ON / RTC_*_OFF, RTC_RD_TIME, RTC_SET_TIME, RTC_ALM_READ/SET, RTC_WKALM_*)
-->

## Summary

The **RTC class** binds a hardware real-time-clock driver (`drivers/rtc/rtc-cmos.c`, `rtc-pl031.c`, `rtc-s3c.c`, …) to the kernel's RTC framework via `struct rtc_device` and the `rtc_class` sysfs class. Per-`rtc_device` carries: an `ops` vtable (read_time / set_time / read_alarm / set_alarm / set_offset / alarm_irq_enable / read_offset / param_get / param_set / ioctl / proc) which the driver fills in; a `features` bitmap (RTC_FEATURE_ALARM, RTC_FEATURE_UPDATE_INTERRUPT, RTC_FEATURE_CORRECTION, ALARM_RES_MINUTE/2S, BACKUP_SWITCH_MODE, NEED_WEEK_DAY); the AIE timer (`aie_timer`) driving the per-alarm IRQ, the UIE rtctimer (`uie_rtctimer`) for once-per-second update IRQ, the PIE hrtimer (`pie_timer`) for periodic IRQ; a `timerqueue` of pending rtc_timer expirations driven by `irqwork`/`rtc_timer_do_work`; a wake_queue (`irq_queue`) feeding `/dev/rtcN` read()/poll(); the embedded `char_dev` cdev that, together with `&rtc->dev`, surfaces `/dev/rtcN` via `cdev_device_add`; per-IDA-allocated ID forming the `rtcN` name. Per-`devm_rtc_allocate_device(dev) + rtc_register_device(rtc)` is the canonical bring-up: allocate, fill `rtc->ops`, set `range_min/max`, register. Per-boot `hctosys` sync: if the registered RTC name matches `CONFIG_RTC_HCTOSYS_DEVICE`, `rtc_hctosys()` reads the clock and `do_settimeofday64`-s the system. Per-PM-suspend/resume: snapshot RTC vs system wallclock at suspend; on resume inject elapsed `sleep_time` into the timekeeper. Critical for: dependable wall-clock recovery, wake-from-suspend by alarm, NTP-disciplined hctosys, container-time alignment.

This Tier-3 covers `drivers/rtc/class.c` (~491 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rtc_device` | per-device descriptor + ops vtable | `RtcDevice` |
| `const struct rtc_class_ops` | per-driver vtable | `RtcClassOps` |
| `rtc_class` (struct class) | per-sysfs class "rtc" | `Rtc::class` |
| `rtc_ida` | per-rtcN id allocator | `Rtc::ida` |
| `rtc_device_release()` | per-refdrop destructor | `Rtc::device_release` |
| `rtc_allocate_device()` | per-alloc + init timers | `Rtc::allocate_device` |
| `devm_rtc_allocate_device()` | per-devres-managed alloc | `Rtc::devm_allocate_device` |
| `devm_rtc_release_device()` | per-devres put_device | `Rtc::devm_release_device` |
| `devm_rtc_unregister_device()` | per-devres unregister | `Rtc::devm_unregister_device` |
| `__devm_rtc_register_device()` | per-register + cdev + proc + hctosys | `Rtc::register_device` |
| `devm_rtc_device_register()` | per-legacy combined register | `Rtc::devm_device_register` |
| `rtc_device_get_id()` | per-of-alias or dynamic IDA | `Rtc::device_get_id` |
| `rtc_device_get_offset()` | per-start-year range remap | `Rtc::device_get_offset` |
| `rtc_hctosys()` | per-boot RTC→system sync | `Rtc::hctosys` |
| `rtc_hctosys_ret` | per-init result global | `Rtc::hctosys_ret` |
| `rtc_suspend()` / `rtc_resume()` | per-PM wallclock snapshot/replay | `Rtc::suspend` / `resume` |
| `rtc_class_dev_pm_ops` | per-class PM ops | `Rtc::class_dev_pm_ops` |
| `rtc_init()` (subsys_initcall) | per-class registration | `Rtc::init` |
| `rtc_timer_init()` (interface.c) | per-rtctimer alarm-timer | shared |
| `rtc_aie_update_irq()` (interface.c) | per-AIE rtctimer hook | shared |
| `rtc_uie_update_irq()` (interface.c) | per-UIE rtctimer hook | shared |
| `rtc_pie_update_irq()` (interface.c) | per-PIE hrtimer hook | shared |
| `rtc_timer_do_work()` (interface.c) | per-irqwork timerqueue expirer | shared |
| `rtc_initialize_alarm()` (interface.c) | per-boot pre-existing alarm load | shared |
| `__rtc_read_alarm()` (interface.c) | per-driver-or-projected alarm read | shared |
| `rtc_dev_prepare()` (dev.c) | per-cdev_init | shared |
| `rtc_dev_init()` (dev.c) | per-alloc chrdev region | shared |
| `rtc_proc_add_device()` / `_del_device()` (proc.c) | per-/proc/driver/rtc | shared |
| `rtc_get_dev_attribute_groups()` (sysfs.c) | per-sysfs attribute groups | shared |
| `to_rtc_device()` | per-container_of | macro |

## Compatibility contract

REQ-1: struct rtc_device fields (relevant to class.c):
- dev: embedded struct device (class = &rtc_class).
- id: int (IDA-allocated; rtc%d name).
- ops: const struct rtc_class_ops *.
- owner: struct module *.
- features: DECLARE_BITMAP(RTC_FEATURE_CNT) — { RTC_FEATURE_ALARM, _UPDATE_INTERRUPT, _CORRECTION, _ALARM_RES_MINUTE, _ALARM_RES_2S, _BACKUP_SWITCH_MODE, _NEED_WEEK_DAY, … }.
- flags: unsigned long — { RTC_NO_CDEV bit }.
- ops_lock: struct mutex — serializes ops invocations + timerqueue mutation.
- irq_lock: spinlock_t — protects irq_data.
- irq_data: unsigned long (IRQ status bits — RTC_IRQF/PF/AF/UF or'd together).
- irq_queue: wait_queue_head_t — /dev/rtcN read()/poll() blocker.
- irqwork: struct work_struct (rtc_timer_do_work).
- aie_timer: struct rtc_timer (AIE alarm).
- uie_rtctimer: struct rtc_timer (UIE update).
- pie_timer: struct hrtimer (PIE periodic).
- pie_enabled: int.
- irq_freq: u32 (default 1).
- max_user_freq: u32 (default 64) — UAPI cap.
- timerqueue: struct timerqueue_head (pending rtc_timer expirations).
- range_min, range_max: time64_t / timeu64_t — driver-declared HW range.
- start_secs, offset_secs, set_start_time, set_offset_nsec: range-remap state.
- char_dev: struct cdev (the /dev/rtcN cdev).
- proc, nvram, … : optional auxiliary.

REQ-2: struct rtc_class_ops vtable (in include/linux/rtc.h):
- ioctl(rtc, cmd, arg): driver-specific ioctls.
- read_time(rtc, &tm): obligatory — converts hardware to rtc_time.
- set_time(rtc, &tm): obligatory.
- read_alarm(rtc, &alm): optional — required for RTC_FEATURE_ALARM.
- set_alarm(rtc, &alm): optional — clear RTC_FEATURE_ALARM if NULL at register.
- proc(rtc, seq): optional /proc augmentation.
- alarm_irq_enable(rtc, enabled): optional.
- read_offset(rtc, &offset) / set_offset(rtc, offset): optional — enable RTC_FEATURE_CORRECTION if set_offset present.
- param_get(rtc, param) / param_set(rtc, param): optional UAPI.

REQ-3: rtc_class:
- .name = "rtc".
- .pm = &rtc_class_dev_pm_ops (only if CONFIG_PM_SLEEP ∧ CONFIG_RTC_HCTOSYS_DEVICE).

REQ-4: rtc_device_release(dev) (destructor on last put):
- rtc = to_rtc_device(dev).
- mutex_lock(&rtc.ops_lock).
- while node = timerqueue_getnext(&rtc.timerqueue): timerqueue_del(node).
- mutex_unlock.
- cancel_work_sync(&rtc.irqwork).
- ida_free(&rtc_ida, rtc.id).
- mutex_destroy(&rtc.ops_lock).
- kfree(rtc).

REQ-5: rtc_allocate_device():
- rtc = kzalloc(sizeof(*rtc), GFP_KERNEL); on failure return NULL.
- device_initialize(&rtc.dev).
- rtc.set_offset_nsec = NSEC_PER_SEC + 5·NSEC_PER_MSEC (default RTC-second-after-write transport).
- rtc.irq_freq = 1; rtc.max_user_freq = 64.
- rtc.dev.class = &rtc_class.
- rtc.dev.groups = rtc_get_dev_attribute_groups() — sysfs.c provides date/time/since_epoch/name/max_user_freq/hctosys/wakealarm/offset/range/{alarms_*}.
- rtc.dev.release = rtc_device_release.
- mutex_init(&rtc.ops_lock); spin_lock_init(&rtc.irq_lock); init_waitqueue_head(&rtc.irq_queue).
- timerqueue_init_head(&rtc.timerqueue); INIT_WORK(&rtc.irqwork, rtc_timer_do_work).
- rtc_timer_init(&rtc.aie_timer, rtc_aie_update_irq, rtc).
- rtc_timer_init(&rtc.uie_rtctimer, rtc_uie_update_irq, rtc).
- hrtimer_setup(&rtc.pie_timer, rtc_pie_update_irq, CLOCK_MONOTONIC, HRTIMER_MODE_REL).
- rtc.pie_enabled = 0.
- set_bit(RTC_FEATURE_ALARM, rtc.features).
- set_bit(RTC_FEATURE_UPDATE_INTERRUPT, rtc.features).
- return rtc.

REQ-6: rtc_device_get_id(dev) → int:
- of_id = -1; if dev.of_node: of_id = of_alias_get_id(dev.of_node, "rtc").
- else if dev.parent.of_node: of_id = of_alias_get_id(dev.parent.of_node, "rtc").
- if of_id ≥ 0: id = ida_alloc_range(&rtc_ida, of_id, of_id, GFP_KERNEL); on fail dev_warn "aliases ID %d not available".
- if id < 0: id = ida_alloc(&rtc_ida, GFP_KERNEL).
- return id (or -ERRNO on failure).

REQ-7: rtc_device_get_offset(rtc):
- if range_min == range_max: return (driver did not declare HW range).
- device_property_read_u32(parent, "start-year", &start_year). On success: rtc.start_secs = mktime64(start_year,1,1,0,0,0); rtc.set_start_time = true.
- if !set_start_time: return (no expansion).
- range_secs = range_max - range_min + 1.
- /* Four cases of {start_secs, range_min, range_max} relation: */
- if (start_secs ≥ 0 ∧ start_secs > range_max) ∨ (start_secs + range_secs - 1 < range_min): offset_secs = start_secs - range_min.
- else if start_secs > range_min: offset_secs = range_secs.
- else if start_secs < range_min: offset_secs = -range_secs.
- else: offset_secs = 0.

REQ-8: devm_rtc_unregister_device(data):
- rtc = data; mutex_lock(&rtc.ops_lock).
- rtc_proc_del_device(rtc).
- if !test_bit(RTC_NO_CDEV, &rtc.flags): cdev_device_del(&rtc.char_dev, &rtc.dev).
- rtc.ops = NULL (so subsequent rtc_class_open users observe gone-state).
- mutex_unlock.

REQ-9: devm_rtc_release_device(res):
- rtc = res; put_device(&rtc.dev) — triggers rtc_device_release when last ref dropped.

REQ-10: devm_rtc_allocate_device(dev) → struct rtc_device *:
- id = rtc_device_get_id(dev); if id < 0: return ERR_PTR(id).
- rtc = rtc_allocate_device(); if !rtc: ida_free(id); return ERR_PTR(-ENOMEM).
- rtc.id = id; rtc.dev.parent = dev.
- devm_add_action_or_reset(dev, devm_rtc_release_device, rtc)?.
- dev_set_name(&rtc.dev, "rtc%d", id)?.
- return rtc.

REQ-11: __devm_rtc_register_device(owner, rtc):
- if !rtc.ops: dev_dbg "no ops set"; return -EINVAL.
- if !rtc.ops.set_alarm: clear_bit(RTC_FEATURE_ALARM, rtc.features) (downgrade).
- if rtc.ops.set_offset: set_bit(RTC_FEATURE_CORRECTION, rtc.features).
- rtc.owner = owner.
- rtc_device_get_offset(rtc).
- err = __rtc_read_alarm(rtc, &alrm); if !err: rtc_initialize_alarm(rtc, &alrm) — schedules aie_timer if a future alarm is HW-persistent.
- rtc_dev_prepare(rtc) — cdev_init(&rtc.char_dev, &rtc_dev_fops); rtc.char_dev.owner = owner.
- err = cdev_device_add(&rtc.char_dev, &rtc.dev): on success dev_dbg "char device (M:N)"; on failure set_bit(RTC_NO_CDEV, &rtc.flags) + dev_warn (device still registered as sysfs node).
- rtc_proc_add_device(rtc) — /proc/driver/rtc symlink for rtc0.
- dev_info "registered as <name>".
- if CONFIG_RTC_HCTOSYS_DEVICE ∧ strcmp(dev_name, CONFIG_RTC_HCTOSYS_DEVICE) == 0: rtc_hctosys(rtc).
- return devm_add_action_or_reset(parent, devm_rtc_unregister_device, rtc).

REQ-12: devm_rtc_device_register(dev, name, ops, owner) [DEPRECATED]:
- rtc = devm_rtc_allocate_device(dev); if IS_ERR: return rtc.
- rtc.ops = ops.
- __devm_rtc_register_device(owner, rtc)?.
- return rtc.

REQ-13: rtc_hctosys(rtc) (CONFIG_RTC_HCTOSYS_DEVICE only):
- tv64 = { tv_nsec = NSEC_PER_SEC >> 1 } (half-second nudge; RTC has 1s granularity).
- err = rtc_read_time(rtc, &tm); on err: dev_err + rtc_hctosys_ret = err; return.
- tv64.tv_sec = rtc_tm_to_time64(&tm).
- if BITS_PER_LONG == 32 ∧ tv64.tv_sec > INT_MAX: err = -ERANGE; record + return.
- err = do_settimeofday64(&tv64).
- dev_info "setting system clock to %ptR UTC (%lld)".
- rtc_hctosys_ret = err.

REQ-14: rtc_hctosys_ret global:
- Initialized to -ENODEV.
- Read by rtc-hctosys-ret sysfs and tied to sysfs hctosys attribute.

REQ-15: rtc_suspend(dev) (CONFIG_PM_SLEEP ∧ CONFIG_RTC_HCTOSYS_DEVICE):
- if timekeeping_rtc_skipsuspend(): return 0 (NTP/PPS owns time).
- if strcmp(dev_name, CONFIG_RTC_HCTOSYS_DEVICE) != 0: return 0 (only system RTC).
- err = rtc_read_time(rtc, &tm); on err pr_debug + return 0.
- ktime_get_real_ts64(&old_system).
- old_rtc.tv_sec = rtc_tm_to_time64(&tm).
- delta = timespec64_sub(old_system, old_rtc).
- delta_delta = timespec64_sub(delta, old_delta).
- if |delta_delta.tv_sec| ≥ 2: old_delta = delta (assume time-correction occurred).
- else: old_system = timespec64_sub(old_system, delta_delta) (compensate drift across resume).
- return 0.

REQ-16: rtc_resume(dev):
- if timekeeping_rtc_skipresume(): return 0.
- rtc_hctosys_ret = -ENODEV.
- if strcmp(dev_name, CONFIG_RTC_HCTOSYS_DEVICE) != 0: return 0.
- ktime_get_real_ts64(&new_system).
- err = rtc_read_time(rtc, &tm); on err pr_debug + return 0.
- new_rtc.tv_sec = rtc_tm_to_time64(&tm); new_rtc.tv_nsec = 0.
- if new_rtc.tv_sec < old_rtc.tv_sec: pr_debug "time travel"; return 0 (negative sleep = clock unreliable).
- sleep_time = timespec64_sub(new_rtc, old_rtc).
- sleep_time = timespec64_sub(sleep_time, timespec64_sub(new_system, old_system)) (subtract kernel runtime around suspend/resume).
- if sleep_time.tv_sec ≥ 0: timekeeping_inject_sleeptime64(&sleep_time).
- rtc_hctosys_ret = 0.

REQ-17: rtc_class_dev_pm_ops:
- SIMPLE_DEV_PM_OPS(rtc_suspend, rtc_resume).
- Compiled to NULL when !PM_SLEEP || !HCTOSYS_DEVICE.

REQ-18: AIE rtctimer (aie_timer):
- Fires when a one-shot HW alarm time matches.
- rtc_aie_update_irq(rtc): increments irq_data with RTC_IRQF | RTC_AF, wakes irq_queue; userspace blocked on read() of /dev/rtcN sees the alarm.

REQ-19: UIE rtctimer (uie_rtctimer):
- Fires every 1 s (update interrupt).
- rtc_uie_update_irq(rtc): irq_data |= RTC_IRQF | RTC_UF; wake irq_queue.

REQ-20: PIE hrtimer (pie_timer):
- Periodic, CLOCK_MONOTONIC, HRTIMER_MODE_REL.
- Frequency = irq_freq (capped by max_user_freq via RTC_IRQP_SET).
- rtc_pie_update_irq(timer): irq_data |= RTC_IRQF | RTC_PF; wake irq_queue; return HRTIMER_RESTART if pie_enabled.

REQ-21: rtc_timer_do_work(work):
- Drains rtc->timerqueue under ops_lock.
- For each expired rtc_timer node: dequeue, call node.func(rtc); if periodic, requeue.
- Invoked by rtc_handle_legacy_irq() / rtc_irq_set_state() / rtc_aie_update_irq() etc. (interface.c).

REQ-22: rtc_initialize_alarm(rtc, &alrm) (interface.c):
- If alrm.enabled ∧ alrm.time is in the future: schedule aie_timer for alrm.time.
- Used at register time to honor pre-existing HW alarms (e.g., wake-from-S5).

REQ-23: __rtc_read_alarm(rtc, &alrm) (interface.c):
- If ops.read_alarm: call it; else if RTC_FEATURE_ALARM cleared: -EINVAL.
- Projects alarm time forward if HW only stores partial (m/h/d) fields.

REQ-24: rtc_dev_prepare(rtc) (dev.c):
- cdev_init(&rtc.char_dev, &rtc_dev_fops).
- rtc.char_dev.owner = owner.
- Reserves the (RTC_DEV_MAJOR, rtc.id) devt.

REQ-25: rtc_dev_init(void) (dev.c, called from rtc_init):
- err = alloc_chrdev_region(&rtc_devt, 0, RTC_DEV_MAX, "rtc").
- pr_err on failure; rtc_devt stays 0 ⇒ no /dev/rtcN created.

REQ-26: rtc_dev_fops (dev.c):
- .read: blocks on irq_queue; copies irq_data to userspace (legacy UIE/AIE/PIE one-shot semantics).
- .poll: irq_queue.
- .ioctl: { RTC_RD_TIME, RTC_SET_TIME, RTC_ALM_READ, RTC_ALM_SET, RTC_WKALM_RD, RTC_WKALM_SET, RTC_AIE_ON, RTC_AIE_OFF, RTC_UIE_ON, RTC_UIE_OFF, RTC_PIE_ON, RTC_PIE_OFF, RTC_IRQP_READ, RTC_IRQP_SET, RTC_EPOCH_READ, RTC_EPOCH_SET, RTC_VL_READ, RTC_VL_CLR, RTC_PARAM_GET, RTC_PARAM_SET, …}.
- .open: takes ops_lock briefly to verify rtc.ops != NULL.
- .release: rtc_dev_ioctl(file, RTC_UIE_OFF, 0) — disables UIE on close.

REQ-27: rtc_get_dev_attribute_groups() (sysfs.c):
- Returns const struct attribute_group ** including: date, time, since_epoch, name, max_user_freq, hctosys, wakealarm, offset, range, alarms_*.
- Each attribute calls into rtc_read_time / rtc_read_alarm / rtc_read_offset.

REQ-28: rtc_proc_add_device(rtc) / rtc_proc_del_device(rtc) (proc.c):
- Only for rtc.id == 0: create /proc/driver/rtc symlink to /sys/class/rtc/rtc0.
- _del removes the symlink.

REQ-29: rtc_init(void) [subsys_initcall]:
- err = class_register(&rtc_class)?.
- rtc_dev_init() — alloc chrdev region.
- return 0.

REQ-30: IDA semantics:
- DT alias "rtc0" / "rtc1" reserved via of_alias_get_id; dev_warn on conflict; fallback to dynamic.
- ida_alloc on rtc_ida; ida_free on release.

REQ-31: set_offset_nsec default = 1·sec + 5·ms — RTC's "next second after write" transport. Drivers can override after devm_rtc_allocate_device.

REQ-32: features at allocation:
- RTC_FEATURE_ALARM set by default; cleared at register if ops.set_alarm == NULL.
- RTC_FEATURE_UPDATE_INTERRUPT set by default; drivers clear if HW lacks 1Hz update IRQ.
- RTC_FEATURE_CORRECTION set at register if ops.set_offset != NULL.

REQ-33: Order at register: ops gate → feature downgrade → range-remap → HW-alarm load → cdev → proc → log → devres unregister-action.

REQ-34: devres unregister/release composition:
- devm_rtc_allocate_device installs devm_rtc_release_device (put_device).
- __devm_rtc_register_device additionally installs devm_rtc_unregister_device.
- On parent detach: devres unwinds in reverse — unregister first (proc_del + cdev_del + ops=NULL), then release (put_device → release → kfree).

## Acceptance Criteria

- [ ] AC-1: devm_rtc_allocate_device returns rtc with class=&rtc_class, refcount=1, name "rtcN".
- [ ] AC-2: Driver assigning ops without set_alarm: register clears RTC_FEATURE_ALARM.
- [ ] AC-3: Driver assigning ops with set_offset: register sets RTC_FEATURE_CORRECTION.
- [ ] AC-4: __rtc_read_alarm at register loads HW alarm; future-dated alarm starts aie_timer.
- [ ] AC-5: Register fails with -EINVAL when ops==NULL.
- [ ] AC-6: cdev_device_add failure sets RTC_NO_CDEV; sysfs registration still proceeds; dev_warn issued.
- [ ] AC-7: dev_name(rtc) == CONFIG_RTC_HCTOSYS_DEVICE triggers rtc_hctosys at register; rtc_hctosys_ret reflects result.
- [ ] AC-8: rtc_hctosys on 32-bit BITS_PER_LONG with tv_sec > INT_MAX returns -ERANGE; system clock not set.
- [ ] AC-9: rtc_suspend snapshots old_system + old_rtc; compensates drift when |delta_delta| < 2s, resets when ≥ 2s.
- [ ] AC-10: rtc_resume injects elapsed RTC delta (minus kernel runtime) via timekeeping_inject_sleeptime64; if RTC went backwards: pr_debug "time travel" and skip.
- [ ] AC-11: Unbind (devres unwind): proc removed, cdev_device_del run, ops cleared, then put_device drops final ref → release → kfree.
- [ ] AC-12: rtc_device_release cancels irqwork synchronously and frees all pending timerqueue nodes under ops_lock.
- [ ] AC-13: aie_timer fires → rtc_aie_update_irq sets RTC_IRQF | RTC_AF in irq_data and wakes irq_queue.
- [ ] AC-14: pie_timer with pie_enabled=1 returns HRTIMER_RESTART; pie_enabled=0 returns HRTIMER_NORESTART.
- [ ] AC-15: rtc_device_get_id with DT alias "rtc0" reserves id=0 in IDA; mismatched second device with same alias falls back to dynamic id and dev_warn.

## Architecture

```
struct RtcDevice {
  dev: Device,              // class = &rtc_class
  id: i32,
  ops: Option<&'static RtcClassOps>,
  owner: *mut Module,
  features: Bitmap<RTC_FEATURE_CNT>,
  flags: AtomicULong,       // RTC_NO_CDEV
  ops_lock: Mutex,
  irq_lock: Spinlock,
  irq_data: AtomicULong,    // RTC_IRQF | RTC_AF | RTC_UF | RTC_PF
  irq_queue: WaitQueueHead,
  irq_freq: u32,
  max_user_freq: u32,
  irqwork: WorkStruct,      // rtc_timer_do_work
  aie_timer: RtcTimer,      // rtc_aie_update_irq
  uie_rtctimer: RtcTimer,   // rtc_uie_update_irq
  pie_timer: Hrtimer,       // rtc_pie_update_irq, CLOCK_MONOTONIC, MODE_REL
  pie_enabled: i32,
  timerqueue: TimerQueueHead,
  range_min: time64_t,
  range_max: timeu64_t,
  start_secs: time64_t,
  offset_secs: time64_t,
  set_start_time: bool,
  set_offset_nsec: u64,     // default NSEC_PER_SEC + 5·NSEC_PER_MSEC
  char_dev: Cdev,           // /dev/rtcN
}

struct RtcClassOps {
  ioctl: Option<fn(&mut RtcDevice, u32, u64) -> i32>,
  read_time: fn(&RtcDevice, &mut RtcTime) -> i32,
  set_time: fn(&mut RtcDevice, &RtcTime) -> i32,
  read_alarm: Option<fn(&RtcDevice, &mut RtcWkalrm) -> i32>,
  set_alarm: Option<fn(&mut RtcDevice, &RtcWkalrm) -> i32>,
  proc: Option<fn(&RtcDevice, &mut SeqFile)>,
  alarm_irq_enable: Option<fn(&mut RtcDevice, bool) -> i32>,
  read_offset: Option<fn(&RtcDevice, &mut i64) -> i32>,
  set_offset: Option<fn(&mut RtcDevice, i64) -> i32>,
  param_get: Option<fn(&RtcDevice, &mut RtcParam) -> i32>,
  param_set: Option<fn(&mut RtcDevice, &RtcParam) -> i32>,
}
```

`Rtc::allocate_device() -> Option<Box<RtcDevice>>`:
1. rtc = kzalloc(RtcDevice). On failure: None.
2. device_initialize(&rtc.dev).
3. rtc.set_offset_nsec = NSEC_PER_SEC + 5·NSEC_PER_MSEC.
4. rtc.irq_freq = 1; rtc.max_user_freq = 64.
5. rtc.dev.class = &rtc_class.
6. rtc.dev.groups = rtc_get_dev_attribute_groups().
7. rtc.dev.release = Rtc::device_release.
8. mutex_init + spin_lock_init + init_waitqueue_head.
9. timerqueue_init_head + INIT_WORK(rtc_timer_do_work).
10. rtc_timer_init(&aie_timer, rtc_aie_update_irq).
11. rtc_timer_init(&uie_rtctimer, rtc_uie_update_irq).
12. hrtimer_setup(&pie_timer, rtc_pie_update_irq, CLOCK_MONOTONIC, HRTIMER_MODE_REL).
13. pie_enabled = 0.
14. set_bit(RTC_FEATURE_ALARM, features).
15. set_bit(RTC_FEATURE_UPDATE_INTERRUPT, features).

`Rtc::devm_allocate_device(parent_dev) -> Result<&RtcDevice>`:
1. id = Rtc::device_get_id(parent_dev)?.
2. rtc = Rtc::allocate_device(); on None: ida_free(id); return Err(-ENOMEM).
3. rtc.id = id; rtc.dev.parent = parent_dev.
4. devm_add_action_or_reset(parent_dev, Rtc::devm_release_device, rtc)?.
5. dev_set_name(&rtc.dev, "rtc%d", id)?.
6. return Ok(rtc).

`Rtc::register_device(owner, rtc) -> Result`:
1. if rtc.ops.is_none(): dev_dbg "no ops set"; return Err(-EINVAL).
2. if rtc.ops.unwrap().set_alarm.is_none(): clear_bit(RTC_FEATURE_ALARM, features).
3. if rtc.ops.unwrap().set_offset.is_some(): set_bit(RTC_FEATURE_CORRECTION, features).
4. rtc.owner = owner.
5. Rtc::device_get_offset(rtc).
6. if Ok(alrm) = Rtc::read_alarm_raw(rtc): rtc_initialize_alarm(rtc, &alrm).
7. rtc_dev_prepare(rtc).
8. if Ok = cdev_device_add(&rtc.char_dev, &rtc.dev): dev_dbg "char device".
   else: set_bit(RTC_NO_CDEV, &rtc.flags); dev_warn.
9. rtc_proc_add_device(rtc).
10. dev_info "registered as %s".
11. if CONFIG_RTC_HCTOSYS_DEVICE matches dev_name: Rtc::hctosys(rtc).
12. devm_add_action_or_reset(parent, Rtc::devm_unregister_device, rtc).

`Rtc::devm_unregister_device(rtc)`:
1. guard(mutex)(&rtc.ops_lock).
2. rtc_proc_del_device(rtc).
3. if !test_bit(RTC_NO_CDEV, &rtc.flags): cdev_device_del(&rtc.char_dev, &rtc.dev).
4. rtc.ops = None.

`Rtc::device_release(dev)` (last put_device):
1. rtc = container_of(dev, RtcDevice, dev).
2. guard(mutex)(&rtc.ops_lock).
3. while node = timerqueue_getnext(&rtc.timerqueue): timerqueue_del(node).
4. unlock.
5. cancel_work_sync(&rtc.irqwork).
6. ida_free(&rtc_ida, rtc.id).
7. mutex_destroy(&rtc.ops_lock).
8. kfree(rtc).

`Rtc::device_get_id(dev) -> Result<i32>`:
1. of_id = -1.
2. if dev.of_node: of_id = of_alias_get_id(dev.of_node, "rtc").
3. else if dev.parent ∧ dev.parent.of_node: of_id = of_alias_get_id(dev.parent.of_node, "rtc").
4. if of_id ≥ 0: id = ida_alloc_range(&rtc_ida, of_id, of_id); on Err: dev_warn "aliases ID %d not available"; fallthrough.
5. if id is invalid: id = ida_alloc(&rtc_ida).
6. return id.

`Rtc::hctosys(rtc)` (CONFIG_RTC_HCTOSYS_DEVICE only):
1. tv64.tv_nsec = NSEC_PER_SEC >> 1.
2. err = rtc_read_time(rtc, &tm). on Err: dev_err + rtc_hctosys_ret = err; return.
3. tv64.tv_sec = rtc_tm_to_time64(&tm).
4. if BITS_PER_LONG == 32 ∧ tv64.tv_sec > INT_MAX: err = -ERANGE; rtc_hctosys_ret = err; return.
5. err = do_settimeofday64(&tv64).
6. dev_info "setting system clock to %ptR UTC (%lld)".
7. rtc_hctosys_ret = err.

`Rtc::suspend(dev)` (CONFIG_PM_SLEEP ∧ CONFIG_RTC_HCTOSYS_DEVICE):
1. if timekeeping_rtc_skipsuspend(): return 0.
2. if dev_name(&rtc.dev) != CONFIG_RTC_HCTOSYS_DEVICE: return 0.
3. err = rtc_read_time(rtc, &tm); on err pr_debug + return 0.
4. ktime_get_real_ts64(&old_system).
5. old_rtc.tv_sec = rtc_tm_to_time64(&tm).
6. delta = old_system - old_rtc.
7. delta_delta = delta - old_delta.
8. if |delta_delta.tv_sec| ≥ 2: old_delta = delta.
9. else: old_system = old_system - delta_delta.

`Rtc::resume(dev)`:
1. if timekeeping_rtc_skipresume(): return 0.
2. rtc_hctosys_ret = -ENODEV.
3. if dev_name(&rtc.dev) != CONFIG_RTC_HCTOSYS_DEVICE: return 0.
4. ktime_get_real_ts64(&new_system).
5. err = rtc_read_time(rtc, &tm); on err pr_debug + return 0.
6. new_rtc.tv_sec = rtc_tm_to_time64(&tm); new_rtc.tv_nsec = 0.
7. if new_rtc.tv_sec < old_rtc.tv_sec: pr_debug "time travel"; return 0.
8. sleep_time = new_rtc - old_rtc.
9. sleep_time -= (new_system - old_system).
10. if sleep_time.tv_sec ≥ 0: timekeeping_inject_sleeptime64(&sleep_time).
11. rtc_hctosys_ret = 0.

`Rtc::device_get_offset(rtc)`:
1. if range_min == range_max: return.
2. if device_property_read_u32(parent, "start-year", &year) == 0:
   - start_secs = mktime64(year, 1, 1, 0, 0, 0).
   - set_start_time = true.
3. if !set_start_time: return.
4. range_secs = range_max - range_min + 1.
5. if (start_secs ≥ 0 ∧ start_secs > range_max) ∨ (start_secs + range_secs - 1 < range_min): offset_secs = start_secs - range_min.
6. else if start_secs > range_min: offset_secs = range_secs.
7. else if start_secs < range_min: offset_secs = -range_secs.
8. else: offset_secs = 0.

`Rtc::init()` (subsys_initcall):
1. class_register(&rtc_class)?.
2. rtc_dev_init() — alloc_chrdev_region for "rtc".
3. return Ok.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ops_lock_held_for_timerqueue_mutation` | INVARIANT | per-device_release: ops_lock held while draining timerqueue. |
| `irqwork_cancel_before_free` | INVARIANT | per-device_release: cancel_work_sync before kfree. |
| `ida_free_paired_with_alloc` | INVARIANT | per-allocate/release: every id from rtc_ida is freed exactly once. |
| `ops_null_after_unregister` | INVARIANT | per-devm_rtc_unregister_device: ops = NULL set under ops_lock — preserves rtc_class_open observers. |
| `register_requires_ops` | INVARIANT | per-__devm_rtc_register_device: !ops ⟹ -EINVAL without side effects. |
| `feature_alarm_downgrade` | INVARIANT | per-register: ops.set_alarm == NULL ⟹ RTC_FEATURE_ALARM cleared before any cdev creation. |
| `feature_correction_upgrade` | INVARIANT | per-register: ops.set_offset != NULL ⟹ RTC_FEATURE_CORRECTION set. |
| `cdev_failure_keeps_device` | INVARIANT | per-register: cdev_device_add Err ⟹ RTC_NO_CDEV set, but proc + sysfs still added. |
| `hctosys_only_on_matching_device` | INVARIANT | per-register: rtc_hctosys called iff dev_name == CONFIG_RTC_HCTOSYS_DEVICE. |
| `hctosys_y2038_guard_on_32bit` | INVARIANT | per-hctosys: BITS_PER_LONG == 32 ∧ tv_sec > INT_MAX ⟹ -ERANGE, do_settimeofday64 not called. |
| `pm_only_for_hctosys_device` | INVARIANT | per-suspend/resume: short-circuit if dev_name != CONFIG_RTC_HCTOSYS_DEVICE. |
| `resume_skips_negative_sleep` | INVARIANT | per-resume: new_rtc < old_rtc ⟹ no inject_sleeptime64. |
| `pie_hrtimer_clock_monotonic_rel` | INVARIANT | per-allocate: pie_timer initialized with CLOCK_MONOTONIC + HRTIMER_MODE_REL. |
| `default_features_set` | INVARIANT | per-allocate: ALARM + UPDATE_INTERRUPT set; CORRECTION not yet (depends on ops). |

### Layer 2: TLA+

`drivers/rtc/class.tla`:
- Per-allocate + per-register + per-unregister + per-hctosys + per-PM-suspend/resume + per-AIE/UIE/PIE timer lifecycle.
- Properties:
  - `safety_id_uniqueness` — at most one rtc_device holds each id in rtc_ida.
  - `safety_ops_published_after_full_init` — register order: features → range → alarm-load → cdev → proc.
  - `safety_no_ops_call_after_unregister` — per-devm_rtc_unregister_device: ops=NULL ⟹ rtc_read_time / rtc_set_alarm return -ENODEV.
  - `safety_hctosys_invoked_iff_match` — rtc_hctosys ⟹ rtc.dev_name == CONFIG_RTC_HCTOSYS_DEVICE.
  - `safety_suspend_resume_pairing` — every rtc_resume preceded by exactly one rtc_suspend on the same dev.
  - `safety_inject_sleeptime_only_when_positive` — timekeeping_inject_sleeptime64 called ⟹ sleep_time ≥ 0.
  - `liveness_register_eventually_creates_devN_or_no_cdev` — per-register: either cdev_device_add succeeds or RTC_NO_CDEV is set.
  - `liveness_release_kfrees_eventually` — per-final-put_device: rtc_device_release runs ∧ kfree(rtc) eventually.
  - `liveness_alarm_irq_wakes_irq_queue` — per-AIE timer fire: irq_data updated and irq_queue wake_up issued.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Rtc::allocate_device` post: irq_freq=1 ∧ max_user_freq=64 ∧ class=&rtc_class ∧ ALARM+UPDATE_INTERRUPT set | `Rtc::allocate_device` |
| `Rtc::devm_allocate_device` post: id is from rtc_ida ∧ name="rtc<id>" ∧ parent set ∧ devres release-action registered | `Rtc::devm_allocate_device` |
| `Rtc::register_device` post: ops != NULL ⟹ features adjusted (ALARM ↔ set_alarm; CORRECTION ↔ set_offset) ∧ proc added ∧ devres unregister-action registered | `Rtc::register_device` |
| `Rtc::devm_unregister_device` post: ops=NULL ∧ proc gone ∧ (cdev gone ∨ RTC_NO_CDEV was set) | `Rtc::devm_unregister_device` |
| `Rtc::device_release` post: timerqueue empty ∧ irqwork canceled ∧ id freed ∧ kfree(rtc) | `Rtc::device_release` |
| `Rtc::hctosys` post: rtc_hctosys_ret == 0 iff do_settimeofday64 succeeded | `Rtc::hctosys` |
| `Rtc::suspend` post: |delta_delta.tv_sec| ≥ 2 ⟹ old_delta := delta; else old_system adjusted | `Rtc::suspend` |
| `Rtc::resume` post: positive sleep_time injected into timekeeping; rtc_hctosys_ret cleared/set | `Rtc::resume` |
| `Rtc::device_get_offset` post: offset_secs matches the four-case piecewise function | `Rtc::device_get_offset` |
| `Rtc::device_get_id` post: returned id ∈ rtc_ida ∧ DT-alias honored when available | `Rtc::device_get_id` |

### Layer 4: Verus/Creusot functional

`Per-driver probe → devm_rtc_allocate_device → driver sets ops + range → __devm_rtc_register_device (features adjust → range remap → __rtc_read_alarm + rtc_initialize_alarm → rtc_dev_prepare → cdev_device_add → rtc_proc_add_device → rtc_hctosys-if-system) → /dev/rtcN serves RTC_RD_TIME / RTC_ALM_SET / RTC_AIE_ON → suspend snapshots → resume injects sleep_time → unbind reverses (devres unwind: devm_rtc_unregister_device → devm_rtc_release_device → rtc_device_release)` semantic equivalence: per-Documentation/admin-guide/rtc.rst + Documentation/driver-api/rtc.rst.

## Hardening

(Inherits row-1 features from `drivers/rtc/00-overview.md` § Hardening.)

RTC-class reinforcement:

- **Per-rtc_device_release acquires ops_lock before draining timerqueue** — defense against per-concurrent rtc_timer_do_work UAF.
- **Per-cancel_work_sync(&irqwork) before kfree** — defense against per-worker-running-after-free.
- **Per-ida_free in release path; ida_alloc on allocate** — defense against per-id leak under register-failure unwind.
- **Per-rtc.ops = NULL under ops_lock at unregister** — defense against per-rtc_class_open consumer racing with rtc.ops dereference.
- **Per-register refuses !ops** — defense against per-NULL-vtable crash during read_time.
- **Per-feature downgrade (ALARM iff set_alarm) at register** — defense against per-uapi advertising unimplemented alarm.
- **Per-cdev_device_add failure: dev_warn but device still sysfs-registered** — defense against per-half-registered-device leak (devres unregister still cleans up).
- **Per-hctosys 32-bit Y2038 guard** — defense against per-time_t overflow at boot.
- **Per-PM-suspend/resume gated on CONFIG_RTC_HCTOSYS_DEVICE name match** — defense against per-N-RTC-fighting-over-system-clock.
- **Per-rtc_resume "time travel" guard (new_rtc < old_rtc)** — defense against per-flaky-RTC injecting bogus negative sleep.
- **Per-rtc_resume kernel-runtime subtraction before inject_sleeptime64** — defense against per-double-count of resume code path.
- **Per-set_offset_nsec default = 1s + 5ms** — defense against per-RTC-write-1s-jitter.
- **Per-DT-alias-id-conflict dev_warn + dynamic fallback** — defense against per-duplicate-rtcN collision.
- **Per-PIE hrtimer CLOCK_MONOTONIC + REL** — defense against per-wallclock-step-induced misfire.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — /dev/rtcN ioctl arg buffers (struct rtc_time, struct rtc_wkalrm, struct rtc_pll_info) and /sys/class/rtc/rtcN/* attrs whitelisted; defense against per-oversized-ioctl-read leaking adjacent slab.
- **PAX_KERNEXEC** — rtc_class_open, rtc_read_time, rtc_set_time, rtc_set_alarm, rtc_irq_handler, rtc_aie_update_irq run W^X.
- **PAX_RANDKSTACK** — per-/dev/rtcN-ioctl and per-RTC-IRQ entry randomize kernel-stack offset; defense against per-RTC-side-channel via stack-prefetch.
- **PAX_REFCOUNT** — struct rtc_device, rtc_class_open file refs saturating refcount_t; defense against per-open-storm refcount overflow.
- **PAX_MEMORY_SANITIZE** — rtc_device, rtc_timer, rtc_wkalrm, rtc_task slabs poison-on-free; defense against per-prior-alarm leak across close/re-open.
- **PAX_UDEREF** — RTC_RD_TIME / RTC_SET_TIME / RTC_ALM_SET / RTC_WKALM_SET / RTC_PIE_ON copy_*_user enforce split user/kernel.
- **PAX_RAP/kCFI** — struct rtc_class_ops vtable (read_time, set_time, read_alarm, set_alarm, alarm_irq_enable, ioctl) CFI-protected.
- **GRKERNSEC_HIDESYM** — rtc_devices list, per-driver rtc_class_ops addresses hidden from /proc/kallsyms.
- **GRKERNSEC_DMESG** — "rtc: hctosys", "rtc%d: alarm rollover" prints restricted.
- **/dev/rtcN CAP_SYS_TIME** — RTC_SET_TIME, RTC_EPOCH_SET, RTC_PARAM_SET (per-driver writable params) require CAP_SYS_TIME; defense against per-unprivileged-clock-skew (replay attacks against time-sensitive auth, breakage of TLS not-before/not-after, breakage of kerberos ticket lifetime).
- **RTC_AIE/PIE/UIE CAP_WAKE_ALARM** — RTC_AIE_ON, RTC_PIE_ON, RTC_UIE_ON, RTC_WKALM_SET (alarm/periodic/update IRQ enable + wake-alarm program) gated on CAP_WAKE_ALARM; defense against per-unprivileged-power-state-manipulation waking a suspended system at attacker-controlled times.
- **Per-DT-alias-id collision dev_warn + fallback** — defense against per-duplicate-rtcN collision.
- **Per-PIE hrtimer CLOCK_MONOTONIC + REL** — defense against per-wallclock-step-induced misfire.
- **Per-alarm time-of-day bounded** — rtc_set_alarm validates rtc_wkalrm.time against rtc_valid_tm; defense against per-out-of-range alarm wedging the driver's set_alarm callback.
- Rationale: /dev/rtcN exposes wallclock setting and wake-from-suspend programming directly to userspace; grsec posture combines CAP_SYS_TIME on time-set, CAP_WAKE_ALARM on alarm/periodic/wake-alarm enables, CFI on rtc_class_ops, usercopy-whitelisting on the rtc_time/rtc_wkalrm slab, and sanitize-on-free of rtc_timer/wkalrm across close.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/rtc/dev.c /dev/rtcN file_operations + RTC_AIE_ON / RTC_PIE_ON / RTC_UIE_ON / RTC_RD_TIME / RTC_SET_TIME / RTC_ALM_SET / RTC_WKALM_SET ioctl dispatch (covered separately if expanded)
- drivers/rtc/interface.c rtc_read_time / rtc_set_time / rtc_set_alarm / rtc_initialize_alarm / rtc_timer_enqueue / rtc_timer_do_work (covered separately if expanded)
- drivers/rtc/sysfs.c attribute groups (date, time, since_epoch, wakealarm, offset, range) (covered separately if expanded)
- drivers/rtc/proc.c /proc/driver/rtc (covered separately if expanded)
- drivers/rtc/nvmem.c RTC NVRAM passthrough (covered separately if expanded)
- drivers/rtc/rtc-cmos.c, rtc-pl031.c, rtc-s3c.c per-hardware drivers (covered separately if expanded)
- kernel/time/timekeeping.c do_settimeofday64 / timekeeping_inject_sleeptime64 (covered in time/timekeeping Tier-3)
- kernel reboot / poweroff path (no shutdown hook exists in class.c at this baseline)
- Implementation code
