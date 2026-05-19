# Tier-3: kernel/panic.c — Kernel panic / oops / WARN core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/panic.c (~1263 lines)
  - include/linux/panic.h
  - include/linux/panic_notifier.h
  - include/linux/bug.h (WARN / WARN_ON / __WARN_FLAGS)
  - include/linux/sys_info.h (SYS_INFO_*, panic_sys_info= parsing)
-->

## Summary

`kernel/panic.c` is the kernel's last-resort termination path. When a fatal condition is detected (BUG, oops escalation, stack-canary failure, NMI fatal, watchdog, deliberate `panic(fmt, ...)` call), `vpanic` is invoked — only the **first** CPU that wins `panic_try_start()`'s atomic cmpxchg on `panic_cpu` proceeds; all others either redirect themselves via `panic_smp_self_stop()` or, in NMI context, `nmi_panic_self_stop(regs)`. The panic CPU then: (1) disables interrupts + preemption, (2) optionally redirects to the `panic_force_cpu=` CPU for crash-kernel needs, (3) emits the message via `pr_emerg("Kernel panic - not syncing: %s")`, (4) optionally `dump_stack()`, (5) invokes `kgdb_panic()`, (6) invokes `__crash_kexec(NULL)` (pre-notifiers by default; post-notifiers if `crash_kexec_post_notifiers=1`), (7) shuts down other CPUs via `panic_other_cpus_shutdown`, (8) runs `panic_notifier_list` chain, (9) prints sys_info per `panic_print` bitmap (`SYS_INFO_TASKS | SYS_INFO_MEM | SYS_INFO_TIMERS | SYS_INFO_LOCKS | SYS_INFO_FTRACE | SYS_INFO_ALL_BT | SYS_INFO_BLOCKED_TASKS | SYS_INFO_ALL_PRINTK_MSG`), (10) kmsg_dump(KMSG_DUMP_PANIC), (11) console_flush_on_panic, (12) optional console-replay, (13) `panic_timeout` second blink-then-reboot via `panic_blink` and `emergency_restart`, otherwise loop forever blinking. `oops_in_progress` is the global counter the bug subsystem increments via `oops_enter` / decrements via `oops_exit`; nesting > 1 disables further dump_stack to avoid recursion. `__warn` + `warn_slowpath_fmt` provide the WARN_ON()/WARN() backend, increment `warn_count`, taint the kernel, and may escalate to panic via `panic_on_warn` or `warn_limit`. Critical for: post-mortem (kdump/kexec), debuggability (oops scrolling on multi-CPU), reboot-on-oops policies, FIPS / NIST / common-criteria-style attestation.

This Tier-3 covers `kernel/panic.c` (~1263 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `panic(fmt, ...)` | per-fatal | `Panic::panic` |
| `vpanic(fmt, va_list)` | per-fatal (va_list) | `Panic::vpanic` |
| `nmi_panic(regs, msg)` | per-NMI-fatal | `Panic::nmi_panic` |
| `check_panic_on_warn(origin)` | per-WARN escalation | `Panic::check_on_warn` |
| `panic_try_start()` | per-CPU-arbitration | `Panic::try_start` |
| `panic_reset()` | per-CPU-arbitration reset | `Panic::reset` |
| `panic_in_progress()` | per-state query | `Panic::in_progress` |
| `panic_on_this_cpu()` | per-state query | `Panic::on_this_cpu` |
| `panic_on_other_cpu()` | per-state query | `Panic::on_other_cpu` |
| `panic_smp_self_stop()` | per-other-CPU stop | `Panic::smp_self_stop` (weak) |
| `nmi_panic_self_stop(regs)` | per-NMI other-CPU stop | `Panic::nmi_smp_self_stop` (weak) |
| `crash_smp_send_stop()` | per-crash-kexec other-CPU stop | `Panic::crash_smp_send_stop` (weak) |
| `panic_other_cpus_shutdown(crash_kexec)` | per-CPU-shutdown | `Panic::other_cpus_shutdown` |
| `panic_trigger_all_cpu_backtrace()` | per-NMI-backtrace | `Panic::trigger_all_cpu_backtrace` |
| `panic_smp_redirect_cpu(target, msg)` | per-IPI/NMI redirect | `Panic::smp_redirect_cpu` (weak) |
| `panic_try_force_cpu(fmt, args)` | per-panic_force_cpu= redirect | `Panic::try_force_cpu` |
| `do_panic_on_target_cpu(info)` | per-CSD callback | `Panic::do_on_target_cpu` |
| `panic_notifier_list` | atomic notifier head | `Panic::NOTIFIER_LIST` |
| `panic_blink` | per-LED blink callback | `Panic::BLINK` |
| `no_blink(state)` | default no-op blink | `Panic::no_blink` |
| `panic_print` | sys_info bitmap | `Panic::PRINT_MASK` |
| `panic_timeout` | reboot delay | `Panic::TIMEOUT` |
| `panic_on_oops` | oops → panic | `Panic::ON_OOPS` |
| `panic_on_warn` | warn → panic | `Panic::ON_WARN` |
| `panic_on_taint` | taint → panic | `Panic::ON_TAINT` |
| `panic_on_taint_nousertaint` | userspace cannot set those flags | `Panic::ON_TAINT_NOUSERTAINT` |
| `warn_count` | atomic counter | `Panic::WARN_COUNT` |
| `warn_limit` | per-warn cap | `Panic::WARN_LIMIT` |
| `pause_on_oops` | per-oops pause seconds | `Panic::PAUSE_ON_OOPS` |
| `oops_in_progress` | global recursion counter | `Panic::OOPS_IN_PROGRESS` (provided by `arch/x86/.../die`) |
| `oops_enter()` / `oops_exit()` | per-oops handler entry/exit | `Panic::oops_enter` / `oops_exit` |
| `oops_may_print()` | per-oops print gate | `Panic::oops_may_print` |
| `do_oops_enter_exit()` | per-pause logic | `Panic::do_oops_enter_exit` |
| `__warn(file, line, caller, taint, regs, args)` | per-WARN backend | `Panic::warn_inner` |
| `warn_slowpath_fmt(file, line, taint, fmt, ...)` | per-WARN entry | `Panic::warn_slowpath_fmt` |
| `__warn_printk(fmt, ...)` | per-WARN-flags entry | `Panic::warn_printk` |
| `__stack_chk_fail()` | per-stack-canary | `Panic::stack_chk_fail` |
| `crash_kexec_post_notifiers` | per-flag | `Panic::CRASH_KEXEC_POST_NOTIFIERS` |
| `panic_cpu` (atomic) | arbitration state | `Panic::CPU` |
| `panic_redirect_cpu` (atomic) | per-redirect arbitration | `Panic::REDIRECT_CPU` |
| `panic_triggering_all_cpu_backtrace` | per-printer enable | `Panic::TRIGGERING_ALL_CPU_BT` |
| `panic_this_cpu_backtrace_printed` | per-state | `Panic::THIS_CPU_BT_PRINTED` |
| `print_tainted()` / `print_tainted_verbose()` | per-banner | `Panic::print_tainted` / `_verbose` |
| `add_taint(flag, lockdep_ok)` | per-set-bit | `Panic::add_taint` |
| `test_taint(flag)` / `get_taint()` | per-query | `Panic::test_taint` / `get_taint` |

## Compatibility contract

REQ-1: Global state:
- `atomic_t panic_cpu = PANIC_CPU_INVALID`: per-arbitration; only one CPU can be panic-cpu at a time.
- `atomic_t panic_redirect_cpu = PANIC_CPU_INVALID`: per-`panic_force_cpu=` redirect arbitration.
- `int panic_timeout = CONFIG_PANIC_TIMEOUT`: per-reboot delay seconds (0 = never, <0 = forever-loop without reboot).
- `int panic_on_oops = IS_ENABLED(CONFIG_PANIC_ON_OOPS)`: per-policy.
- `int panic_on_warn`: per-policy.
- `unsigned long panic_on_taint`: per-taint-flag-bitmap that converts taint into panic.
- `bool panic_on_taint_nousertaint`: per-proc-taint setting forbids userspace from setting flags in `panic_on_taint`.
- `bool crash_kexec_post_notifiers`: per-policy (default false: kexec first; true: notifiers first).
- `unsigned long panic_print`: per-sys-info bitmap.
- `static int pause_on_oops`, `static int pause_on_oops_flag`, `static DEFINE_SPINLOCK(pause_on_oops_lock)`: per-CPU oops-pause arbitration.
- `static unsigned int warn_limit`: per-warn cap (panic after N WARNs).
- `static atomic_t warn_count = ATOMIC_INIT(0)`: per-counter.
- `bool panic_triggering_all_cpu_backtrace`: per-printer-bypass during BT.
- `static bool panic_this_cpu_backtrace_printed`: per-state.
- `long (*panic_blink)(int state)`: per-LED-blink callback (default `no_blink` returning 0).
- `static int panic_force_cpu = -1`: per-boot-param target CPU.
- `static char *panic_force_buf`: per-redirect formatted-message buffer (PANIC_MSG_BUFSZ).
- `ATOMIC_NOTIFIER_HEAD(panic_notifier_list)`: per-panic callback chain (atomic).
- `static unsigned long tainted_mask`: per-bitmap of TAINT_*.
- `int oops_in_progress`: per-arch-provided global counter (incremented by `oops_enter`).

REQ-2: panic_try_start() — atomic CPU arbitration:
- old_cpu = PANIC_CPU_INVALID; this_cpu = raw_smp_processor_id().
- return atomic_try_cmpxchg(&panic_cpu, &old_cpu, this_cpu).
- /* Caller proceeds with panic only if returns true */

REQ-3: panic_reset() — clear arbitration:
- atomic_set(&panic_cpu, PANIC_CPU_INVALID).
- /* Used by callers that abort the panic-in-progress state (rare) */

REQ-4: panic_in_progress() / panic_on_this_cpu() / panic_on_other_cpu():
- panic_in_progress(): unlikely(atomic_read(&panic_cpu) != PANIC_CPU_INVALID).
- panic_on_this_cpu(): unlikely(atomic_read(&panic_cpu) == raw_smp_processor_id()).
- panic_on_other_cpu(): panic_in_progress() ∧ !panic_on_this_cpu().

REQ-5: nmi_panic(regs, msg):
- if panic_try_start(): panic("%s", msg).
- else if panic_on_other_cpu(): nmi_panic_self_stop(regs).
- /* If panic_on_this_cpu(): we were already panicking — return (recursion case) */

REQ-6: panic_try_force_cpu(fmt, args) (SMP+CRASH_DUMP only; else inline false):
- if panic_force_cpu < 0: return false.
- if this_cpu == panic_force_cpu: return false (already on target).
- if !cpu_online(panic_force_cpu): warn + return false.
- if panic_in_progress(): return false.
- /* Single redirect winner via panic_redirect_cpu cmpxchg */
- if !atomic_try_cmpxchg(&panic_redirect_cpu, &old_cpu (=INVALID), this_cpu): return false.
- /* Format message into panic_force_buf (or static fallback) */
- console_verbose(); bust_spinlocks(1).
- pr_emerg("panic: Redirecting from CPU %d to CPU %d ...").
- /* Per-DEBUG_BUGVERBOSE pre-redirect dump (only if !test_taint(TAINT_DIE) ∧ oops_in_progress ≤ 1) */
- if panic_smp_redirect_cpu(panic_force_cpu, msg) != 0:
  - atomic_set(&panic_redirect_cpu, PANIC_CPU_INVALID).
  - warn + return false.
- return true.

REQ-7: panic_smp_redirect_cpu(target_cpu, msg) (__weak):
- /* Default: smp_call_function_single_async(target, &panic_csd) where csd.func = do_panic_on_target_cpu */
- Architectures with NMI support may override with NMI-based delivery.

REQ-8: do_panic_on_target_cpu(info):
- panic("%s", (char *)info).

REQ-9: vpanic(fmt, args) — the canonical panic body:
- _crash_kexec_post_notifiers = crash_kexec_post_notifiers.
- if panic_on_warn: panic_on_warn = 0 (avoid recursive panic from WARN in panic path).
- local_irq_disable(); preempt_disable_notrace().
- if panic_try_force_cpu(fmt, args):
  - set_cpu_online(smp_processor_id(), false).
  - panic_smp_self_stop().  /* noreturn */
- if panic_try_start(): /* proceed */
- else if panic_on_other_cpu(): panic_smp_self_stop().  /* noreturn */
- console_verbose(); bust_spinlocks(1).
- len = vscnprintf(buf, PANIC_MSG_BUFSZ, fmt, args); strip trailing '\n'.
- pr_emerg("Kernel panic - not syncing: %s\n", buf).
- /* Per-redirect skip: if we are the redirect target, skip stack dump */
- else if test_taint(TAINT_DIE) ∨ oops_in_progress > 1: panic_this_cpu_backtrace_printed = true (no fresh dump).
- else if CONFIG_DEBUG_BUGVERBOSE: dump_stack(); panic_this_cpu_backtrace_printed = true.
- kgdb_panic(buf).
- if !_crash_kexec_post_notifiers: __crash_kexec(NULL).
- panic_other_cpus_shutdown(_crash_kexec_post_notifiers).
- printk_legacy_allow_panic_sync().
- atomic_notifier_call_chain(&panic_notifier_list, 0, buf).
- sys_info(panic_print).
- kmsg_dump_desc(KMSG_DUMP_PANIC, buf).
- if _crash_kexec_post_notifiers: __crash_kexec(NULL).
- console_unblank().
- debug_locks_off().
- console_flush_on_panic(CONSOLE_FLUSH_PENDING).
- if (panic_print & SYS_INFO_PANIC_CONSOLE_REPLAY) ∨ panic_console_replay: console_flush_on_panic(CONSOLE_REPLAY_ALL).
- if !panic_blink: panic_blink = no_blink.
- /* Reboot phase */
- if panic_timeout > 0:
  - pr_emerg("Rebooting in %d seconds..\n", panic_timeout).
  - for i in 0..(panic_timeout * 1000) step PANIC_TIMER_STEP (=100ms):
    - touch_nmi_watchdog().
    - if i ≥ i_next: i += panic_blink(state ^= 1); i_next = i + 3600 / PANIC_BLINK_SPD.
    - mdelay(PANIC_TIMER_STEP).
- if panic_timeout != 0:
  - if panic_reboot_mode != REBOOT_UNDEFINED: reboot_mode = panic_reboot_mode.
  - emergency_restart().  /* noreturn */
- /* sparc: enable Stop-A. s390: disabled_wait. */
- pr_emerg("---[ end Kernel panic - not syncing: %s ]---\n", buf).
- suppress_printk = 1.
- console_flush_on_panic(CONSOLE_FLUSH_PENDING); nbcon_atomic_flush_unsafe().
- local_irq_enable().
- forever-loop with watchdog touch + blink + mdelay.

REQ-10: panic(fmt, ...):
- va_start; vpanic(fmt, args); va_end.  /* noreturn */

REQ-11: panic_other_cpus_shutdown(crash_kexec):
- if panic_print & SYS_INFO_ALL_BT: panic_trigger_all_cpu_backtrace().
- if !crash_kexec: smp_send_stop().
- else: crash_smp_send_stop().

REQ-12: panic_trigger_all_cpu_backtrace():
- panic_triggering_all_cpu_backtrace = true.
- if panic_this_cpu_backtrace_printed: trigger_allbutcpu_cpu_backtrace(raw_smp_processor_id()).
- else: trigger_all_cpu_backtrace().
- panic_triggering_all_cpu_backtrace = false.

REQ-13: panic_smp_self_stop() (__weak __noreturn):
- while (1) cpu_relax().

REQ-14: nmi_panic_self_stop(regs) (__weak __noreturn):
- panic_smp_self_stop().

REQ-15: crash_smp_send_stop() (__weak):
- static int cpus_stopped; if cpus_stopped: return; smp_send_stop(); cpus_stopped = 1.

REQ-16: panic_blink:
- Default `no_blink(state) -> 0`.
- May be overridden (by drivers like keyboard LED to blink Caps-Lock).
- Called once per ~ (3600 / PANIC_BLINK_SPD = 200ms) interval during reboot countdown + post-reboot infinite loop.
- Return value is added to `i` (effectively a wait-time accumulator).

REQ-17: check_panic_on_warn(origin):
- if panic_on_warn: panic("%s: panic_on_warn set ...", origin).
- limit = READ_ONCE(warn_limit).
- if atomic_inc_return(&warn_count) ≥ limit ∧ limit > 0: panic("%s: system warned too often (kernel.warn_limit is %d)", origin, limit).

REQ-18: __warn(file, line, caller, taint, regs, args):
- nbcon_cpu_emergency_enter().
- disable_trace_on_warning().
- pr_warn("WARNING: %s:%d at %pS, CPU#%d: %s/%d\n", file?, line?, caller, raw_smp_processor_id(), current.comm, current.pid).
- if args: vprintk(args.fmt, args.args).
- print_modules().
- if regs: show_regs(regs).
- check_panic_on_warn("kernel").
- if !regs: dump_stack().
- print_irqtrace_events(current).
- print_oops_end_marker() — `pr_warn("---[ end trace %016llx ]---", 0ULL)`.
- trace_error_report_end(ERROR_DETECTOR_WARN, caller).
- add_taint(taint, LOCKDEP_STILL_OK).
- nbcon_cpu_emergency_exit().

REQ-19: warn_slowpath_fmt(file, line, taint, fmt, ...) (CONFIG_BUG ∧ !__WARN_FLAGS):
- rcu = warn_rcu_enter().
- pr_warn(CUT_HERE) — `pr_warn("------------[ cut here ]------------\n")`.
- if !fmt: __warn(file, line, __builtin_return_address(0), taint, NULL, NULL).
- else: args.fmt = fmt; va_start; __warn(file, line, __builtin_return_address(0), taint, NULL, &args); va_end.
- warn_rcu_exit(rcu).

REQ-20: __warn_printk(fmt, ...) (CONFIG_BUG ∧ __WARN_FLAGS):
- /* Architecture supplies bug-table-driven WARN; __warn_printk only emits message and taints */
- rcu = warn_rcu_enter().
- pr_warn(CUT_HERE).
- va_start; vprintk(fmt, args); va_end.
- warn_rcu_exit(rcu).

REQ-21: oops_enter() / oops_exit() — per-oops gating:
- oops_enter():
  - nbcon_cpu_emergency_enter().
  - tracing_off(); debug_locks_off().
  - do_oops_enter_exit().
  - if sysctl_oops_all_cpu_backtrace: trigger_all_cpu_backtrace().
- oops_exit():
  - do_oops_enter_exit().
  - print_oops_end_marker().
  - nbcon_cpu_emergency_exit().
  - kmsg_dump(KMSG_DUMP_OOPS).

REQ-22: do_oops_enter_exit() — pause_on_oops arbitration:
- if !pause_on_oops: return.
- spin_lock_irqsave(&pause_on_oops_lock, flags).
- if pause_on_oops_flag == 0: pause_on_oops_flag = 1 (this CPU prints).
- else:
  - if !spin_counter (this CPU does the counting):
    - spin_counter = pause_on_oops.
    - do: spin_unlock; spin_msec(MSEC_PER_SEC); spin_lock; while (--spin_counter).
    - pause_on_oops_flag = 0.
  - else (a different CPU is counting): while spin_counter: spin_unlock; spin_msec(1); spin_lock.
- spin_unlock_irqrestore.

REQ-23: oops_may_print(): pause_on_oops_flag == 0.

REQ-24: panic_print masks (SYS_INFO_* — moved from `panic_print` to `panic_sys_info=` cmdline + sysctl):
- `SYS_INFO_ALL_PRINTK_MSG` — bit-0: dump entire printk ring.
- `SYS_INFO_TASKS` — show_state (`SysRq-t`).
- `SYS_INFO_MEM` — show_mem.
- `SYS_INFO_TIMERS` — sysrq_timer_list_show.
- `SYS_INFO_LOCKS` — debug_show_all_locks.
- `SYS_INFO_FTRACE` — ftrace_dump.
- `SYS_INFO_ALL_BT` — trigger_all_cpu_backtrace (handled in panic_other_cpus_shutdown).
- `SYS_INFO_BLOCKED_TASKS` — show_state(TASK_UNINTERRUPTIBLE).
- `SYS_INFO_PANIC_CONSOLE_REPLAY` — console_flush_on_panic(REPLAY_ALL).
- Deprecation: legacy `panic_print=` sysctl writes invoke `panic_print_deprecated()` pr_info_once.

REQ-25: __stack_chk_fail():
- /* gcc -fstack-protector entry on canary mismatch */
- instrumentation_begin(); flags = user_access_save().
- panic("stack-protector: Kernel stack is corrupted in: %pB", __builtin_return_address(0)).  /* noreturn — return path never executes */
- user_access_restore(flags); instrumentation_end().

REQ-26: Boot params / sysctl plumbing:
- `core_param(panic, panic_timeout, int, 0644)`.
- `core_param(pause_on_oops, pause_on_oops, int, 0644)`.
- `core_param(panic_on_warn, panic_on_warn, int, 0644)`.
- `core_param(crash_kexec_post_notifiers, crash_kexec_post_notifiers, bool, 0644)`.
- `core_param(panic_console_replay, panic_console_replay, bool, 0644)`.
- `early_param("panic_force_cpu", panic_force_cpu_setup)`.
- `__setup("panic_sys_info=", setup_panic_sys_info)` — parses comma-separated `tasks,mem,locks,ftrace,...` to `panic_print` bitmap.
- sysctl `kern_panic_table[]`: oops_all_cpu_backtrace, tainted, panic, panic_on_oops, panic_print (deprecated), panic_on_warn, warn_limit, panic_on_stackoverflow (x86_32/parisc only), panic_sys_info.
- proc_taint: writer must have CAP_SYS_ADMIN; panic_on_taint_nousertaint blocks userspace from setting flags in panic_on_taint; sets bits via add_taint().

REQ-27: warn_count sysfs:
- `/sys/kernel/warn_count` shows `atomic_read(&warn_count)`.

REQ-28: panic_notifier_list:
- ATOMIC_NOTIFIER_HEAD; per-handler signature `int (*)(struct notifier_block *, unsigned long, void *)` with priority order via `atomic_notifier_chain_register`.
- Callback chain runs **after** `panic_other_cpus_shutdown` and **before** `kmsg_dump` (or after, with `crash_kexec_post_notifiers`).

## Acceptance Criteria

- [ ] AC-1: First call to panic_try_start on a CPU returns true; concurrent call from another CPU returns false.
- [ ] AC-2: Second panic from a CPU after first acquired panic_cpu: panic_on_this_cpu() true; recursion handled.
- [ ] AC-3: nmi_panic from non-panic-CPU when panic_on_other_cpu(): calls nmi_panic_self_stop (noreturn).
- [ ] AC-4: vpanic with crash_kexec_post_notifiers=0: __crash_kexec invoked before atomic_notifier_call_chain.
- [ ] AC-5: vpanic with crash_kexec_post_notifiers=1: atomic_notifier_call_chain runs before __crash_kexec.
- [ ] AC-6: panic_timeout > 0: emergency_restart() reached after timeout * 1000 ms blink-loop.
- [ ] AC-7: panic_timeout == 0: enters forever-loop with touch_softlockup_watchdog + blink + mdelay.
- [ ] AC-8: panic_timeout < 0: emergency_restart not called; forever-loop.
- [ ] AC-9: check_panic_on_warn: panic_on_warn=1 ⇒ panic.
- [ ] AC-10: check_panic_on_warn: warn_count reaches warn_limit ⇒ panic.
- [ ] AC-11: __warn with regs: show_regs called, dump_stack skipped.
- [ ] AC-12: __warn without regs: dump_stack called.
- [ ] AC-13: __stack_chk_fail: panic with format containing __builtin_return_address(0).
- [ ] AC-14: panic_print & SYS_INFO_ALL_BT set: panic_trigger_all_cpu_backtrace called from panic_other_cpus_shutdown.
- [ ] AC-15: panic_print & SYS_INFO_PANIC_CONSOLE_REPLAY: console_flush_on_panic(CONSOLE_REPLAY_ALL) called.
- [ ] AC-16: panic_force_cpu= configured AND this_cpu != panic_force_cpu AND target online: panic redirects (IPI/NMI) and current CPU stops via panic_smp_self_stop.
- [ ] AC-17: panic_force_cpu= but target offline: redirect skipped (continues on current CPU with warning).

## Architecture

```
struct Panic {                        // module-level statics
  CPU: AtomicI32,                     // panic_cpu, init PANIC_CPU_INVALID
  REDIRECT_CPU: AtomicI32,            // panic_redirect_cpu
  NOTIFIER_LIST: AtomicNotifierHead<NotifierBlock>,
  TIMEOUT: AtomicI32,                 // panic_timeout (mutable via core_param)
  ON_OOPS: AtomicI32,
  ON_WARN: AtomicI32,
  ON_TAINT: AtomicUsize,
  ON_TAINT_NOUSERTAINT: AtomicBool,
  CRASH_KEXEC_POST_NOTIFIERS: AtomicBool,
  PRINT_MASK: AtomicUsize,            // SYS_INFO_*
  WARN_COUNT: AtomicI32,
  WARN_LIMIT: AtomicU32,
  PAUSE_ON_OOPS: AtomicI32,
  PAUSE_ON_OOPS_FLAG: AtomicI32,
  PAUSE_ON_OOPS_LOCK: SpinLock,
  CONSOLE_REPLAY: AtomicBool,
  BLINK: AtomicPtr<fn(i32) -> i64>,   // panic_blink (default no_blink)
  FORCE_CPU: AtomicI32,               // -1 = disabled
  FORCE_BUF: AtomicPtr<u8>,           // PANIC_MSG_BUFSZ buffer
  TRIGGERING_ALL_CPU_BT: AtomicBool,
  THIS_CPU_BT_PRINTED: AtomicBool,
  TAINTED_MASK: AtomicUsize,
  OOPS_IN_PROGRESS: AtomicI32,        // bumped/decremented by oops_enter/exit
}

const PANIC_TIMER_STEP: u32 = 100;    // ms
const PANIC_BLINK_SPD: u32 = 18;
const PANIC_MSG_BUFSZ: usize = 1024;
const PANIC_CPU_INVALID: i32 = -1;
```

`Panic::try_start() -> bool`:
1. old_cpu = PANIC_CPU_INVALID.
2. this_cpu = raw_smp_processor_id().
3. Panic::CPU.compare_exchange(old_cpu, this_cpu, AcqRel, Acquire).is_ok().

`Panic::reset()`:
1. Panic::CPU.store(PANIC_CPU_INVALID, Release).

`Panic::in_progress() -> bool`:
1. Panic::CPU.load(Acquire) != PANIC_CPU_INVALID.

`Panic::on_this_cpu() -> bool`:
1. Panic::CPU.load(Acquire) == raw_smp_processor_id().

`Panic::on_other_cpu() -> bool`:
1. Panic::in_progress() ∧ !Panic::on_this_cpu().

`Panic::nmi_panic(regs, msg)`:
1. if Panic::try_start(): Panic::panic_str(msg). /* noreturn */
2. else if Panic::on_other_cpu(): Panic::nmi_smp_self_stop(regs). /* noreturn */
3. /* on_this_cpu: re-entry — return; outer panic continues */

`Panic::vpanic(fmt, args) -> !`:
1. let post_notifiers = Panic::CRASH_KEXEC_POST_NOTIFIERS.load(Acquire).
2. if Panic::ON_WARN.swap(0, AcqRel) != 0 { /* prevent nested panic on WARN in panic path */ }
3. local_irq_disable(); preempt_disable_notrace().
4. if Panic::try_force_cpu(fmt, args):
   - set_cpu_online(smp_processor_id(), false).
   - Panic::smp_self_stop().  /* noreturn */
5. match Panic::try_start() {
     true => { /* proceed */ }
     false if Panic::on_other_cpu() => Panic::smp_self_stop(),  /* noreturn */
     false => { /* on_this_cpu — re-entry */ }
   }
6. console_verbose(); bust_spinlocks(1).
7. let len = vscnprintf(&mut buf, PANIC_MSG_BUFSZ, fmt, args); if buf[len-1] == '\n': buf[len-1] = '\0'.
8. pr_emerg("Kernel panic - not syncing: {}\n", buf).
9. if Panic::REDIRECT_CPU.load(Acquire) != PANIC_CPU_INVALID ∧ Panic::FORCE_CPU.load(Acquire) == raw_smp_processor_id():
   - pr_emerg("panic: Redirected from CPU {}, skipping stack dump.\n").
10. else if test_taint(TAINT_DIE) ∨ Panic::OOPS_IN_PROGRESS.load(Acquire) > 1:
    - Panic::THIS_CPU_BT_PRINTED.store(true, Release).
11. else if cfg!(CONFIG_DEBUG_BUGVERBOSE):
    - dump_stack(); Panic::THIS_CPU_BT_PRINTED.store(true, Release).
12. kgdb_panic(buf).
13. if !post_notifiers: __crash_kexec(NULL).
14. Panic::other_cpus_shutdown(post_notifiers).
15. printk_legacy_allow_panic_sync().
16. atomic_notifier_call_chain(&Panic::NOTIFIER_LIST, 0, buf).
17. sys_info(Panic::PRINT_MASK.load(Acquire)).
18. kmsg_dump_desc(KMSG_DUMP_PANIC, buf).
19. if post_notifiers: __crash_kexec(NULL).
20. console_unblank().
21. debug_locks_off().
22. console_flush_on_panic(CONSOLE_FLUSH_PENDING).
23. if Panic::PRINT_MASK.load(Acquire) & SYS_INFO_PANIC_CONSOLE_REPLAY ∨ Panic::CONSOLE_REPLAY.load(Acquire):
    - console_flush_on_panic(CONSOLE_REPLAY_ALL).
24. if Panic::BLINK.load(Acquire).is_null(): Panic::BLINK.store(no_blink, Release).
25. let timeout = Panic::TIMEOUT.load(Acquire).
26. if timeout > 0:
    - pr_emerg("Rebooting in {} seconds..\n", timeout).
    - blink-loop for timeout*1000 ms (PANIC_TIMER_STEP=100ms; touch_nmi_watchdog each step).
27. if timeout != 0:
    - if panic_reboot_mode != REBOOT_UNDEFINED: reboot_mode = panic_reboot_mode.
    - emergency_restart().  /* noreturn */
28. /* sparc / s390 quirks */
29. pr_emerg("---[ end Kernel panic - not syncing: {} ]---\n", buf).
30. suppress_printk = 1.
31. console_flush_on_panic(CONSOLE_FLUSH_PENDING); nbcon_atomic_flush_unsafe().
32. local_irq_enable().
33. loop { touch_softlockup_watchdog(); blink+= panic_blink(state^=1); mdelay(PANIC_TIMER_STEP); }

`Panic::panic(fmt, ...) -> !`:
1. va_start; Panic::vpanic(fmt, args); va_end.

`Panic::other_cpus_shutdown(crash_kexec)`:
1. if Panic::PRINT_MASK & SYS_INFO_ALL_BT: Panic::trigger_all_cpu_backtrace().
2. if !crash_kexec: smp_send_stop().
3. else: crash_smp_send_stop().

`Panic::trigger_all_cpu_backtrace()`:
1. Panic::TRIGGERING_ALL_CPU_BT.store(true, Release).
2. if Panic::THIS_CPU_BT_PRINTED.load(Acquire):
   - trigger_allbutcpu_cpu_backtrace(raw_smp_processor_id()).
3. else: trigger_all_cpu_backtrace().
4. Panic::TRIGGERING_ALL_CPU_BT.store(false, Release).

`Panic::check_on_warn(origin)`:
1. if Panic::ON_WARN.load(Acquire) != 0: Panic::panic("{}: panic_on_warn set ...\n", origin).
2. let limit = Panic::WARN_LIMIT.load(Acquire).
3. let count = Panic::WARN_COUNT.fetch_add(1, AcqRel) + 1.
4. if count ≥ limit ∧ limit != 0:
   - Panic::panic("{}: system warned too often (kernel.warn_limit is {})", origin, limit).

`Panic::warn_inner(file, line, caller, taint, regs, args)`:
1. nbcon_cpu_emergency_enter().
2. disable_trace_on_warning().
3. pr_warn("WARNING: ...").
4. if Some(args): vprintk(args.fmt, args.args).
5. print_modules().
6. if Some(regs): show_regs(regs).
7. Panic::check_on_warn("kernel").
8. if regs.is_none(): dump_stack().
9. print_irqtrace_events(current).
10. Panic::print_oops_end_marker().
11. trace_error_report_end(ERROR_DETECTOR_WARN, caller).
12. Panic::add_taint(taint, LOCKDEP_STILL_OK).
13. nbcon_cpu_emergency_exit().

`Panic::oops_enter()`:
1. nbcon_cpu_emergency_enter().
2. tracing_off(); debug_locks_off().
3. Panic::do_oops_enter_exit().
4. if Panic::OOPS_ALL_CPU_BACKTRACE.load(Acquire): trigger_all_cpu_backtrace().

`Panic::oops_exit()`:
1. Panic::do_oops_enter_exit().
2. Panic::print_oops_end_marker().
3. nbcon_cpu_emergency_exit().
4. kmsg_dump(KMSG_DUMP_OOPS).

`Panic::stack_chk_fail()`:
1. instrumentation_begin(); let flags = user_access_save().
2. Panic::panic("stack-protector: Kernel stack is corrupted in: {:pB}", __builtin_return_address(0)).  /* noreturn */

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `panic_try_start_exclusive` | INVARIANT | per-REQ-2: only one CPU wins cmpxchg per panic_cpu transition. |
| `vpanic_irqs_disabled` | INVARIANT | per-REQ-9: local_irq_disable + preempt_disable_notrace before any other action. |
| `vpanic_post_notifiers_order` | INVARIANT | per-REQ-9: crash_kexec_post_notifiers=true ⇒ notifier chain runs before __crash_kexec. |
| `vpanic_pre_notifiers_order` | INVARIANT | per-REQ-9: crash_kexec_post_notifiers=false ⇒ __crash_kexec runs before notifier chain. |
| `panic_blink_non_null` | INVARIANT | per-REQ-16: panic_blink replaced with no_blink before reboot countdown. |
| `force_cpu_redirect_single_winner` | INVARIANT | per-REQ-6: panic_redirect_cpu cmpxchg admits one redirector. |
| `check_on_warn_panics_when_limit_reached` | INVARIANT | per-REQ-17: warn_count ≥ warn_limit ⇒ panic. |
| `warn_inner_adds_taint` | INVARIANT | per-REQ-18: __warn adds taint after dump_stack + check_panic_on_warn. |
| `stack_chk_fail_calls_panic` | INVARIANT | per-REQ-25: __stack_chk_fail calls panic before any return path. |
| `oops_pause_arbitration_single_printer` | INVARIANT | per-REQ-22: pause_on_oops_flag==1 ⇒ exactly one CPU prints. |

### Layer 2: TLA+

`kernel/panic.tla`:
- States per CPU: {Normal, EnteringPanic, OnPanicCPU, SelfStopped, Rebooting, Looping}.
- Globals: panic_cpu, panic_redirect_cpu, panic_print, panic_timeout, crash_kexec_post_notifiers, warn_count.
- Actions: panic_try_start, panic_try_force_cpu, smp_send_stop, atomic_notifier_call_chain, __crash_kexec, emergency_restart, check_panic_on_warn, oops_enter/exit.
- Properties:
  - `safety_single_panic_cpu` — per-time: at most one CPU has panic_cpu = its-id.
  - `safety_all_other_cpus_stopped` — per-panic-CPU-active: all other CPUs eventually in SelfStopped.
  - `safety_kexec_ordering` — per-crash_kexec_post_notifiers=false: __crash_kexec strictly precedes notifier chain.
  - `safety_kexec_ordering_post` — per-crash_kexec_post_notifiers=true: notifier chain strictly precedes __crash_kexec.
  - `safety_warn_count_monotonic` — warn_count never decreases (no reset path on panic path).
  - `safety_redirect_at_most_once` — per-REQ-6: redirect target wins panic_redirect_cpu cmpxchg at most once.
  - `liveness_timeout_pos_reboots` — panic_timeout > 0 ⇒ eventually emergency_restart.
  - `liveness_timeout_zero_loops_forever` — panic_timeout == 0 ⇒ no emergency_restart; eventually Looping.
  - `liveness_timeout_neg_no_reboot` — panic_timeout < 0 ⇒ no emergency_restart.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Panic::try_start` post: ret ⇒ Panic::CPU == this_cpu | `Panic::try_start` |
| `Panic::vpanic` post: never returns; emergency_restart called iff timeout != 0 | `Panic::vpanic` |
| `Panic::nmi_panic` post: panicked-or-stopped; never resumes original NMI | `Panic::nmi_panic` |
| `Panic::check_on_warn` post: panic_on_warn=1 ⇒ noreturn | `Panic::check_on_warn` |
| `Panic::warn_inner` post: taint added, warn_count incremented (via check_on_warn), trace event emitted | `Panic::warn_inner` |
| `Panic::other_cpus_shutdown` post: all-but-this CPU stopped via smp_send_stop or crash_smp_send_stop | `Panic::other_cpus_shutdown` |
| `Panic::oops_enter` post: tracing off, debug_locks off | `Panic::oops_enter` |
| `Panic::stack_chk_fail` post: noreturn | `Panic::stack_chk_fail` |

### Layer 4: Verus/Creusot functional

`Per-panic flow: vpanic → CPU-arbitration → message + dump_stack → kgdb_panic → kexec/notifier → kmsg_dump → console-flush → reboot-or-loop` semantic equivalence: per `Documentation/admin-guide/kernel-parameters.txt` (panic=, panic_on_warn=, panic_print=, panic_sys_info=, crash_kexec_post_notifiers=, panic_force_cpu=), `Documentation/admin-guide/sysctl/kernel.rst` (kernel.panic, panic_on_oops, panic_on_warn, warn_limit, panic_on_taint).

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

panic-core reinforcement:

- **Per-panic_cpu cmpxchg arbitration** — defense against per-multi-CPU concurrent-panic deadlock.
- **Per-panic_redirect_cpu cmpxchg** — defense against per-double-redirect race.
- **Per-panic_on_warn cleared on entry** — defense against per-WARN-in-panic recursive panic.
- **Per-IRQ + preempt disabled at entry** — defense against per-IRQ-while-panicking re-entry.
- **Per-debug_locks_off** — defense against per-locking-recursion in printk path.
- **Per-test_taint(TAINT_DIE) ∨ oops_in_progress > 1 ⇒ skip dump_stack** — defense against per-nested-oops dump_stack recursion.
- **Per-kgdb_panic before SMP stop** — defense against per-debugger-loses-other-CPU-context.
- **Per-crash_smp_send_stop static cpus_stopped** — defense against per-double-stop in crash path.
- **Per-cpus_stopped sentinel** — defense against per-recursive smp_send_stop.
- **Per-suppress_printk after end-trace marker** — defense against per-late-printk-scroll obscuring panic banner.
- **Per-touch_nmi_watchdog / touch_softlockup_watchdog in blink loop** — defense against per-watchdog-bite-during-panic-wait.
- **Per-warn_limit enforced atomically** — defense against per-warn-spam DoS.
- **Per-panic_on_taint_nousertaint** — defense against per-userspace-injected-taint triggering panic.
- **Per-proc_taint CAP_SYS_ADMIN required** — defense against per-unprivileged-taint manipulation.
- **Per-panic_print=ALL_BT runs BEFORE smp_send_stop** — defense against per-stopped-CPUs unable to send BT.
- **Per-emergency_restart honors panic_reboot_mode** — defense against per-wrong-reboot-method on panic.
- **Per-__crash_kexec ordering documented** — defense against per-kdump-loses-notifier-data (or notifier-corrupts-kdump).

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy.
- **PAX_KERNEXEC** — W^X for any executable mapping.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization.
- **PAX_REFCOUNT** — saturating refcount on subsystem structs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for sensitive allocations.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on vtables.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding.
- **GRKERNSEC_DMESG** — syslog restriction.
- **panic_notifier chain under PAX_RAP** — each `notifier_call` is invoked via kCFI-signed indirect call; an attacker who corrupts the notifier list with a function pointer to a gadget triggers a CFI fault instead of executing the gadget at panic time (the most reliable arbitrary-code moment).
- **panic_on_oops=1 default** — converts any oops into a hard panic, denying the attacker the "oops, retry the exploit" iteration loop.
- **oops_count saturating under PAX_REFCOUNT** — cannot wrap to zero to evade `oops_limit`.
- **panic_print bitmask CAP_SYS_ADMIN-gated** — even with /proc/sys writable, only init-user-ns admin can flip the dump-task-state / dump-memory bits that leak pointers.
- **panic_blink under PAX_RAP** — the architecture-supplied blink callback (often a function pointer set early) checked for kCFI signature before invocation.
- **GRKERNSEC_HIDESYM sanitizes panic banner** — register dumps and stack pointers in the panic banner masked for unprivileged log readers via the dmesg restriction.
- **smp_send_stop ordering preserved** — panic_print runs before stopping CPUs, but stop is non-cancellable; an attacker cannot defer the stop to keep a backdoor CPU live.

Per-doc rationale: panic is the kernel's last-chance code path — it runs with interrupts off, locks bypassed, and notifier callbacks invoked in undefined order. That makes it the prime target for an attacker who has corrupted a function pointer somewhere and wants the kernel to execute it deterministically. PAX_RAP/kCFI on the panic_notifier chain and on `panic_blink` neutralizes the most common gadget path, PAX_REFCOUNT prevents oops_count rollover from defeating the iteration limit, and GRKERNSEC_HIDESYM keeps the panic banner from leaking the symbols an attacker needs for the next attempt.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- arch/*/kernel/dumpstack.c per-arch dump_stack + stack_trace_save (covered in `arch/dumpstack.md` Tier-3)
- arch/*/kernel/traps.c die() / die_nmi() / oops_begin / oops_end (covered in `arch/traps.md` Tier-3)
- kernel/kexec_core.c + __crash_kexec (covered in `kexec.md` Tier-3)
- kernel/printk/printk.c console_flush_on_panic + kmsg_dump (covered in `printk/printk.md` Tier-3)
- kernel/printk/nbcon.c nbcon_atomic_flush_unsafe (covered in `printk/nbcon.md` Tier-3)
- kernel/sys_info.c sys_info(panic_print) dispatch (covered in `sys_info.md` Tier-3)
- kernel/reboot.c emergency_restart + panic_reboot_mode (covered in `reboot.md` Tier-3)
- kernel/debug/kdb (covered in `kdb.md` Tier-3 if expanded)
- kernel/watchdog.c touch_nmi_watchdog + touch_softlockup_watchdog (covered in `watchdog.md` Tier-3)
- lib/bug.c BUG() + report_bug() (covered in `lib/bug.md` Tier-3)
- include/asm-generic/bug.h __WARN_FLAGS (per-arch bug table)
- Implementation code
