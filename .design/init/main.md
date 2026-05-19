# Tier-3: init/main.c — start_kernel and the boot driver

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: init/00-overview.md
upstream-paths:
  - init/main.c (~1734 lines)
  - include/linux/start_kernel.h
  - include/linux/init.h (initcall levels, __setup, early_param)
  - include/linux/init_syscalls.h
-->

## Summary

`init/main.c` is the C-level boot driver. After arch-specific assembly hands control to `start_kernel()`, this file orchestrates every subsystem init in a fixed order, then spawns PID 1 (`kernel_init`) and PID 2 (`kthreadd`) before the boot CPU descends into `cpu_idle`. Per-`start_kernel` runs single-threaded with IRQs disabled at entry, brings up cpu/mm/sched/RCU/timers/IRQs in lock-step, then `rest_init` flips on the scheduler. Per-`kernel_init` (PID 1) waits for `kthreadd`, runs `kernel_init_freeable` (SMP-init, pre-SMP initcalls, `smp_init`, `sched_init_smp`, `do_basic_setup` → `do_initcalls` (8 levels), initramfs wait, `prepare_namespace`), frees init memory, marks rodata, then exec's userspace `init`. Per-`do_initcalls` walks `__initcall{0..7}_start` linker arrays calling each `do_one_initcall(fn)` and parses level-scoped command-line params. Per-`run_init_process` tries the explicit `init=`, then `CONFIG_DEFAULT_INIT`, then `/sbin/init`, `/etc/init`, `/bin/init`, `/bin/sh`; panics if all fail. Critical for: deterministic boot ordering, observable `dmesg` sequence, and userspace handover.

This Tier-3 covers `init/main.c` (~1734 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `start_kernel()` | per-boot main driver | `Init::start_kernel` |
| `rest_init()` | per-boot final stage (spawns PID 1, PID 2, idles) | `Init::rest_init` |
| `kernel_init(void *)` | per-PID-1 kthread | `Init::kernel_init` |
| `kernel_init_freeable()` | per-PID-1 freeable phase | `Init::kernel_init_freeable` |
| `do_basic_setup()` | per-driver/initcall driver | `Init::do_basic_setup` |
| `do_pre_smp_initcalls()` | per-`early` initcalls | `Init::do_pre_smp_initcalls` |
| `do_initcalls()` | per-8-level driver | `Init::do_initcalls` |
| `do_initcall_level(level, cmdline)` | per-level driver | `Init::do_initcall_level` |
| `do_one_initcall(fn)` | per-fn invoker | `Init::do_one_initcall` |
| `initcall_blacklist` / `initcall_blacklisted` | per-blacklist | `Init::initcall_blacklist` |
| `do_ctors()` | per-CONFIG_CONSTRUCTORS dispatch | `Init::do_ctors` |
| `run_init_process(name)` | per-exec userspace init | `Init::run_init_process` |
| `try_to_run_init_process(name)` | per-exec wrapper | `Init::try_to_run_init_process` |
| `setup_command_line(cl)` | per-cmdline buffer setup | `Init::setup_command_line` |
| `parse_early_param()` / `parse_early_options(cl)` | per-`early_param` dispatch | `Init::parse_early_param` |
| `do_early_param(param, val, ...)` | per-early-param matcher | `Init::do_early_param` |
| `unknown_bootoption(param, val, ...)` | per-unknown param → init/env | `Init::unknown_bootoption` |
| `set_init_arg(param, val, ...)` | per-`--` init arg | `Init::set_init_arg` |
| `obsolete_checksetup(line)` | per-`__setup` legacy match | `Init::obsolete_checksetup` |
| `repair_env_string(p, v)` | per-`NUL→=` repair | `Init::repair_env_string` |
| `setup_boot_config()` / `exit_boot_config()` | per-bootconfig | `Init::setup_boot_config` |
| `get_boot_config_from_initrd(*sz)` | per-bootconfig discovery | `Init::get_boot_config_from_initrd` |
| `xbc_make_cmdline(key)` / `xbc_snprint_cmdline(buf,sz,root)` | per-bootconfig → cmdline | `Init::xbc_make_cmdline` |
| `bootconfig_params(p,v,...)` / `warn_bootconfig(str)` | per-cmdline scan | shared |
| `print_kernel_cmdline(cl)` | per-`dmesg` cmdline echo | `Init::print_kernel_cmdline` |
| `print_unknown_bootoptions()` | per-`dmesg` unknown-opts | `Init::print_unknown_bootoptions` |
| `console_on_rootfs()` | per-`/dev/console` fd 0/1/2 | `Init::console_on_rootfs` |
| `mark_readonly()` | per-rodata-lock | `Init::mark_readonly` |
| `free_initmem()` | per-`__init` discard | `Init::free_initmem` |
| `set_debug_rodata(str)` | per-`rodata=on/off` parse | `Init::set_debug_rodata` |
| `set_reset_devices(str)` | per-`reset_devices` parse | shared |
| `init_setup(str)` / `rdinit_setup(str)` | per-`init=` / `rdinit=` parse | shared |
| `debug_kernel(str)` / `quiet_kernel(str)` / `loglevel(str)` | per-loglevel | shared |
| `initcall_blacklist(str)` | per-`initcall_blacklist=` parse | shared |
| `system_state` (enum SYSTEM_*) | per-boot state | shared global |
| `early_boot_irqs_disabled` | per-IRQ-state flag | shared global |
| `boot_command_line[]` / `saved_command_line` / `static_command_line` / `extra_command_line` / `extra_init_args` | per-cmdline buffers | shared globals |
| `loops_per_jiffy` | per-BogoMIPS | shared global |
| `initcall_debug` | per-`initcall_debug` flag | shared |
| `argv_init[]` / `envp_init[]` | per-init exec argv/envp | shared globals |
| `cad_pid` | per-Ctrl-Alt-Del pid | shared global |
| `late_time_init` | per-arch late-timer hook | shared fn-ptr |
| `kthreadd_done` | per-PID-1 / PID-2 sync | static completion |

## Compatibility contract

REQ-1: `start_kernel` orchestration order — observable via `dmesg` when CONFIG_PRINTK_TIME=y:
1. `set_task_stack_end_magic(&init_task)` — per-stack-overflow canary on idle task.
2. `smp_setup_processor_id()` — per-arch BSP id.
3. `debug_objects_early_init()` — per-debugobjects early bring-up.
4. `init_vmlinux_build_id()` — per-`/sys/kernel/notes` build-id parse.
5. `cgroup_init_early()` — per-cgroup css boot.
6. `local_irq_disable(); early_boot_irqs_disabled = true`.
7. `boot_cpu_init()` — per-cpu_online/active/possible/present mask for BSP.
8. `page_address_init()` — per-page→virtual table.
9. `pr_notice("%s", linux_banner)` — first dmesg line: `Linux version <ver> ...`.
10. `setup_arch(&command_line)` — per-arch e820/memory-map/cmdline-import.
11. `mm_core_init_early()` — per-memblock-stage allocator.
12. `jump_label_init()`, `static_call_init()`.
13. `early_security_init()` — per-LSM early hooks.
14. `setup_boot_config()` — per-bootconfig parse from initrd or embedded.
15. `setup_command_line(command_line)` — per-`saved/static_command_line` build.
16. `setup_nr_cpu_ids()`, `setup_per_cpu_areas()`.
17. `smp_prepare_boot_cpu()`.
18. `early_numa_node_init()`, `boot_cpu_hotplug_init()`.
19. `print_kernel_cmdline(saved_command_line)` — per-`Kernel command line: ...` dmesg.
20. `parse_early_param()` — per-`early_param`-flagged `__setup`s.
21. `parse_args("Booting kernel", static_command_line, __start___param, __stop___param-__start___param, -1, -1, NULL, &unknown_bootoption)` — per-`core_param`/`module_param`.
22. `print_unknown_bootoptions()`.
23. `parse_args("Setting init args", after_dashes, NULL, 0, -1, -1, NULL, set_init_arg)` if `--` present.
24. `parse_args("Setting extra init args", extra_init_args, ...)` if bootconfig provided init args.
25. `random_init_early(command_line)`.
26. `setup_log_buf(0)`.
27. `vfs_caches_init_early()` — per-dentry/inode hash pre-alloc.
28. `sort_main_extable()`.
29. `trap_init()` — per-arch IDT.
30. `mm_core_init()` — per-page-allocator/slab. (covers `build_all_zonelists`, `page_alloc_init` (legacy split), `mem_init`, `kmem_cache_init`, `vmalloc_init`)
31. `maple_tree_init()`.
32. `poking_init()` — per-text-poking pages.
33. `ftrace_init()`.
34. `early_trace_init()`.
35. `sched_init()` — per-task `init_task` runqueue setup.
36. `WARN(!irqs_disabled(), ...); local_irq_disable()`.
37. `radix_tree_init()`.
38. `housekeeping_init()`.
39. `workqueue_init_early()`.
40. `rcu_init()`; `kvfree_rcu_init()`.
41. `trace_init()`.
42. `if (initcall_debug) initcall_debug_enable()`.
43. `context_tracking_init()`.
44. `early_irq_init(); init_IRQ()`.
45. `tick_init(); rcu_init_nohz(); timers_init(); srcu_init(); hrtimers_init(); softirq_init()`.
46. `vdso_setup_data_pages()`.
47. `timekeeping_init(); time_init()`.
48. `random_init()`.
49. `kfence_init(); boot_init_stack_canary()`.
50. `perf_event_init(); profile_init(); call_function_init()`.
51. `early_boot_irqs_disabled = false; local_irq_enable()`.
52. `kmem_cache_init_late()`.
53. `console_init()`.
54. `if (panic_later) panic("Too many boot %s vars at \`%s\`", panic_later, panic_param)`.
55. `lockdep_init(); locking_selftest()`.
56. CONFIG_BLK_DEV_INITRD: if `initrd_start && !initrd_below_start_ok && page_to_pfn(...) < min_low_pfn`: `initrd_start = 0` + `pr_crit("initrd overwritten ...")`.
57. `setup_per_cpu_pageset(); numa_policy_init(); acpi_early_init()`.
58. `if (late_time_init) late_time_init()`.
59. `sched_clock_init(); calibrate_delay()` — per-BogoMIPS.
60. `arch_cpu_finalize_init()`.
61. `pid_idr_init()` — per-`pid_idr` (was `idr_init_cache`).
62. `anon_vma_init(); thread_stack_cache_init(); cred_init(); fork_init(); proc_caches_init(); uts_ns_init(); time_ns_init(); key_init(); security_init(); dbg_late_init(); net_ns_init()`.
63. `vfs_caches_init()` — per-mount-points + rootfs registration (covers former `mnt_init`).
64. `pagecache_init(); signals_init(); seq_file_init(); proc_root_init(); nsfs_init(); pidfs_init()`.
65. `cpuset_init(); mem_cgroup_init(); cgroup_init(); taskstats_init_early(); delayacct_init()`.
66. `acpi_subsystem_init(); arch_post_acpi_subsys_init()`.
67. `kcsan_init()`.
68. `rest_init()` — never returns from `start_kernel`'s POV.

REQ-2: `rest_init()`:
- `rcu_scheduler_starting()`.
- `pid = user_mode_thread(kernel_init, NULL, CLONE_FS)` — PID 1.
- `tsk = find_task_by_pid_ns(pid, &init_pid_ns); tsk->flags |= PF_NO_SETAFFINITY; set_cpus_allowed_ptr(tsk, cpumask_of(smp_processor_id()))`.
- `numa_default_policy()`.
- `pid = kernel_thread(kthreadd, NULL, NULL, CLONE_FS | CLONE_FILES)` — PID 2.
- `kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns)`.
- `system_state = SYSTEM_SCHEDULING`.
- `complete(&kthreadd_done)` — unblocks `kernel_init`.
- `schedule_preempt_disabled()` — boot CPU yields.
- `cpu_startup_entry(CPUHP_ONLINE)` — idle loop; noreturn.

REQ-3: `kernel_init(unused)`:
- `wait_for_completion(&kthreadd_done)`.
- `kernel_init_freeable()`.
- `async_synchronize_full()`.
- `system_state = SYSTEM_FREEING_INITMEM`.
- `kprobe_free_init_mem(); ftrace_free_init_mem(); kgdb_free_init_mem(); exit_boot_config(); free_initmem(); mark_readonly()`.
- `pti_finalize()` — per-PTI page-table commit.
- `system_state = SYSTEM_RUNNING`.
- `numa_default_policy()`.
- `rcu_end_inkernel_boot()`.
- `do_sysctl_args()` — per-`__setup` aliased to sysctl.
- Per-`ramdisk_execute_command` (default `/init`): `run_init_process(ramdisk_execute_command)`; if 0 return.
- Per-`execute_command` (set by `init=`): `run_init_process(execute_command)`; if 0 return; else `panic("Requested init %s failed (error %d).")`.
- Per-`CONFIG_DEFAULT_INIT[0] != '\0'`: `run_init_process(CONFIG_DEFAULT_INIT)`; if 0 return.
- Fall-through: try `/sbin/init`, `/etc/init`, `/bin/init`, `/bin/sh` via `try_to_run_init_process` — first success returns 0.
- Otherwise `panic("No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance.")`.

REQ-4: `kernel_init_freeable()`:
- `gfp_allowed_mask = __GFP_BITS_MASK` — full GFP now permitted.
- `set_mems_allowed(node_states[N_MEMORY])`.
- `cad_pid = get_pid(task_pid(current))`.
- `smp_prepare_cpus(setup_max_cpus)`.
- `workqueue_init()`.
- `init_mm_internals()` — per-vmstat/zoneinfo.
- `do_pre_smp_initcalls()` — invokes `__initcall_start..__initcall0_start` (the `early_initcall` level).
- `lockup_detector_init()`.
- `smp_init()` — per-AP bring-up via cpuhp; brings up all secondary CPUs.
- `sched_init_smp()` — per-domain build.
- `workqueue_init_topology(); async_init(); padata_init(); page_alloc_init_late()`.
- `do_basic_setup()` → `cpuset_init_smp(); ksysfs_init(); driver_init(); init_irq_proc(); do_ctors(); do_initcalls()`.
- `kunit_run_all_tests()`.
- `wait_for_initramfs()` — block until initramfs unpack (rootfs_initcall) complete.
- `console_on_rootfs()` — `filp_open("/dev/console", O_RDWR, 0)`; `init_dup` × 3; `fput`.
- `int ramdisk_command_access = init_eaccess(ramdisk_execute_command)`.
- If non-zero: warn (if `rdinit=` set), zero `ramdisk_execute_command`, `prepare_namespace()` — per-do_mounts path (legacy real-root mount).
- `integrity_load_keys()`.

REQ-5: `do_pre_smp_initcalls()`:
- `do_trace_initcall_level("early")`.
- For `fn = __initcall_start; fn < __initcall0_start; fn++`: `do_one_initcall(initcall_from_entry(fn))`.

REQ-6: `do_initcalls()`:
- `len = saved_command_line_len + 1; command_line = kzalloc(len, GFP_KERNEL)`; panic on alloc fail.
- For `level = 0; level < 8; level++`:
  - `strcpy(command_line, saved_command_line)`.
  - `do_initcall_level(level, command_line)`.
- `kfree(command_line)`.

REQ-7: `do_initcall_level(level, command_line)`:
- `parse_args(initcall_level_names[level], command_line, __start___param, __stop___param - __start___param, level, level, NULL, ignore_unknown_bootoption)`.
- `do_trace_initcall_level(initcall_level_names[level])`.
- For `fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++`: `do_one_initcall(initcall_from_entry(fn))`.

REQ-8: 8 initcall levels (must match upstream linker-array ordering):
| Index | Name | Section |
|---|---|---|
| 0 | `pure` | `__initcall0_start` |
| 1 | `core` | `__initcall1_start` |
| 2 | `postcore` | `__initcall2_start` |
| 3 | `arch` | `__initcall3_start` |
| 4 | `subsys` | `__initcall4_start` |
| 5 | `fs` | `__initcall5_start` |
| 6 | `device` | `__initcall6_start` |
| 7 | `late` | `__initcall7_start` |
End sentinel `__initcall_end`. Early section (before `__initcall0_start`) is the `__initcall_start..__initcall0_start` range — invoked by `do_pre_smp_initcalls` before SMP.

REQ-9: `do_one_initcall(fn)`:
- `count = preempt_count()`.
- If `initcall_blacklisted(fn)`: return `-EPERM`.
- `do_trace_initcall_start(fn)`.
- `ret = fn()`.
- `do_trace_initcall_finish(fn, ret)`.
- If `preempt_count() != count`: `sprintf(msgbuf, "preemption imbalance "); preempt_count_set(count)`.
- If `irqs_disabled()`: append `"disabled interrupts "`; `local_irq_enable()`.
- `WARN(msgbuf[0], "initcall %pS returned with %s\n", fn, msgbuf)`.
- `add_latent_entropy()`.
- return `ret`.

REQ-10: `initcall_blacklist`:
- `initcall_blacklist=fn1,fn2,...` parsed via `__setup("initcall_blacklist=", initcall_blacklist)`.
- `initcall_blacklisted(fn)` resolves `fn` via `sprint_symbol_no_offset`, strips `[module]`, matches against list.
- Requires CONFIG_KALLSYMS; otherwise warn on cmdline.

REQ-11: `run_init_process(init_filename)`:
- `argv_init[0] = init_filename`.
- `pr_info("Run %s as init process\n", init_filename)`.
- `pr_debug` arguments + environment.
- return `kernel_execve(init_filename, argv_init, envp_init)`.

REQ-12: `try_to_run_init_process(init_filename)`:
- `ret = run_init_process(init_filename)`.
- If `ret && ret != -ENOENT`: `pr_err("Starting init: %s exists but couldn't execute it (error %d)\n", init_filename, ret)`.
- return `ret`.

REQ-13: Userspace init search order (must match upstream exactly):
1. `ramdisk_execute_command` (default `/init`, override `rdinit=`).
2. `execute_command` (from `init=`; on failure: panic — no fall-through).
3. `CONFIG_DEFAULT_INIT` (if non-empty).
4. `/sbin/init` → `/etc/init` → `/bin/init` → `/bin/sh`.
5. `panic("No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance.")`.

REQ-14: Command-line parsing pipeline:
- `boot_command_line[COMMAND_LINE_SIZE]` — untouched copy from bootloader (saved by arch code).
- `saved_command_line` — for `/proc/cmdline`; built in `setup_command_line` with `extra_command_line` prepended and `extra_init_args` appended after ` -- `.
- `static_command_line` — destructive parse buffer (`parse_args` mutates in place).
- `extra_command_line` — from bootconfig `kernel.*` keys.
- `extra_init_args` — from bootconfig `init.*` keys.
- `parse_early_param()` → `parse_early_options(tmp_cmdline)` → `do_early_param(param, val, ...)` walks `__setup_start..__setup_end` for `p->early == true`.
- `parse_args("Booting kernel", static_command_line, __start___param, ..., -1, -1, NULL, &unknown_bootoption)` — covers `core_param` / `module_param`.
- `unknown_bootoption` — handles `BOOT_IMAGE=`, `kexec`, sysctl-alias (`sysctl_is_alias(param)`), obsolete `__setup` (`obsolete_checksetup(param)`), module params (containing `.`), env (`X=Y`), and finally appends to `argv_init`.
- `parse_args("Setting init args", after_dashes, ..., NULL, set_init_arg)` — params after `--` go to init.
- Per-level `parse_args(initcall_level_names[level], ..., level, level, NULL, ignore_unknown_bootoption)` during `do_initcall_level`.

REQ-15: `setup_command_line(command_line)`:
- `xlen = strlen(extra_command_line)` if set.
- `extra_init_args = strim(extra_init_args); ilen = strlen(extra_init_args) + 4` for ` -- ` if set.
- `len = xlen + strlen(boot_command_line) + ilen + 1`; `saved_command_line = memblock_alloc_or_panic(len, SMP_CACHE_BYTES)`.
- `len = xlen + strlen(command_line) + 1`; `static_command_line = memblock_alloc_or_panic(len, SMP_CACHE_BYTES)`.
- Prepend `extra_command_line` to both; copy `boot_command_line` into `saved_command_line + xlen`; copy `command_line` into `static_command_line + xlen`.
- If `ilen`: if bootconfig provided `initargs_offs`, splice extra init args at offset; else append ` -- ` + `extra_init_args` to `saved_command_line`.
- `saved_command_line_len = strlen(saved_command_line)`.

REQ-16: `setup_boot_config()`:
- `data = get_boot_config_from_initrd(&size)`: scan last 4 bytes of `initrd_end` for `BOOTCONFIG_MAGIC`, validate size + checksum (`xbc_calc_checksum`), trim `initrd_end`.
- If no initrd bootconfig: `data = xbc_get_embedded_bootconfig(&size)`.
- `parse_args("bootconfig", tmp_cmdline, NULL, 0, 0, 0, NULL, bootconfig_params)` — sets `bootconfig_found` if `bootconfig` keyword in cmdline.
- If `!bootconfig_found && !CONFIG_BOOT_CONFIG_FORCE`: return.
- `xbc_init(data, size, &msg, &pos)`; on error pr_err with msg+pos.
- `xbc_get_info(&ret, NULL)`; `pr_info("Load bootconfig: %ld bytes %d nodes\n", ...)`.
- `extra_command_line = xbc_make_cmdline("kernel")`.
- `extra_init_args = xbc_make_cmdline("init")`.

REQ-17: Initcall debug:
- `initcall_debug` boot param (`initcall_debug=Y` via `core_param`).
- `initcall_debug_enable()` (TRACEPOINTS_ENABLED) registers `trace_initcall_start_cb`, `trace_initcall_finish_cb`, `trace_initcall_level_cb`.
- Outputs `calling  %pS @ %i`, `initcall %pS returned %d after %lld usecs`, `entering initcall level: %s`.

REQ-18: `system_state` transitions:
- SYSTEM_BOOTING (initial 0).
- SYSTEM_SCHEDULING — set in `rest_init` just before `complete(&kthreadd_done)`.
- SYSTEM_FREEING_INITMEM — set in `kernel_init` before `free_initmem()`.
- SYSTEM_RUNNING — set in `kernel_init` after `pti_finalize`.
- (SYSTEM_HALT, SYSTEM_POWER_OFF, SYSTEM_RESTART, SYSTEM_SUSPEND — set elsewhere.)

## Acceptance Criteria

- [ ] AC-1: `dmesg` first line matches `"Linux version <ver> ..."` byte-for-byte vs. upstream on the same `linux_banner` build inputs.
- [ ] AC-2: `dmesg` subsystem-init message ordering (every `pr_notice`/`pr_info` emitted from REQ-1 step list) is topologically equivalent to upstream — covered by an integration test that diffs `dmesg | grep ...` against an upstream golden file.
- [ ] AC-3: `cat /proc/cmdline` returns `saved_command_line`, including bootconfig-derived `extra_command_line` prefix and `-- extra_init_args` suffix.
- [ ] AC-4: `init=/path/to/X` causes `run_init_process(X)`; on failure (non-zero return), kernel panics with `"Requested init %s failed (error %d)."`.
- [ ] AC-5: `rdinit=/path/to/X` causes `run_init_process(X)` as the first attempt in `kernel_init`.
- [ ] AC-6: With no `init=`/`rdinit=` and `/init` missing, kernel attempts `/sbin/init`, `/etc/init`, `/bin/init`, `/bin/sh` in order; first success becomes PID 1.
- [ ] AC-7: With no working init anywhere, kernel panics with `"No working init found.  Try passing init= option to kernel. ..."`.
- [ ] AC-8: All 8 initcall levels (`pure`/`core`/`postcore`/`arch`/`subsys`/`fs`/`device`/`late`) plus the pre-SMP `early` level invoke initcalls in linker-array order; `initcall_debug` traces each.
- [ ] AC-9: `initcall_blacklist=fn` prevents `fn` from running (returns `-EPERM`); requires CONFIG_KALLSYMS.
- [ ] AC-10: `do_one_initcall` emits WARN if the initcall left preempt-count imbalanced or IRQs disabled.
- [ ] AC-11: PID 1 (`init`) and PID 2 (`kthreadd`) are spawned by `rest_init` in that order; PID 1 has `PF_NO_SETAFFINITY` and is pinned to the boot CPU at spawn.
- [ ] AC-12: `system_state` transitions BOOTING → SCHEDULING → FREEING_INITMEM → RUNNING in the order specified by REQ-18.
- [ ] AC-13: `mark_readonly()` flushes module init free work and (CONFIG_STRICT_KERNEL_RWX + `rodata_enabled`) marks rodata RO and runs `rodata_test()`; `rodata=off` cmdline disables.
- [ ] AC-14: `console_on_rootfs()` opens `/dev/console` O_RDWR and dup's it to fds 0, 1, 2 for the init process.
- [ ] AC-15: Bootconfig embedded in initrd is detected by `BOOTCONFIG_MAGIC` trailer, validated by `xbc_calc_checksum`, parsed into `extra_command_line` (`kernel.*` keys) and `extra_init_args` (`init.*` keys).

## Architecture

```
struct InitState {
  early_boot_irqs_disabled: AtomicBool,    // observed by lockdep, sched, RCU
  system_state: AtomicU8,                  // SYSTEM_*
  boot_command_line: [u8; COMMAND_LINE_SIZE], // untouched copy from arch
  saved_command_line: Option<KString>,     // /proc/cmdline source
  static_command_line: Option<KString>,    // destructive parse buffer
  extra_command_line: Option<KString>,     // from bootconfig kernel.*
  extra_init_args: Option<KString>,        // from bootconfig init.*
  bootconfig_found: bool,
  initargs_offs: usize,
  execute_command: Option<KString>,        // init=
  ramdisk_execute_command: KString,        // rdinit= (default "/init")
  ramdisk_execute_command_set: bool,
  argv_init: [Option<KString>; MAX_INIT_ARGS + 2], // ["init", NULL, ...]
  envp_init: [Option<KString>; MAX_INIT_ENVS + 2], // ["HOME=/", "TERM=linux", NULL, ...]
  panic_later: Option<&'static str>,
  panic_param: Option<&'static str>,
  initcall_debug: bool,
  loops_per_jiffy: u64,
  cad_pid: Option<*Pid>,
  late_time_init: Option<fn()>,
  kthreadd_done: Completion,
  initcall_blacklist: List<KString>,
}

const INITCALL_LEVELS: [&str; 8] = [
  "pure", "core", "postcore", "arch", "subsys", "fs", "device", "late",
];
```

`Init::start_kernel() -> !`:
1. `set_task_stack_end_magic(&INIT_TASK)`.
2. `arch::smp_setup_processor_id()`.
3. `debug_objects_early_init()`; `init_vmlinux_build_id()`.
4. `cgroup_init_early()`.
5. `arch::local_irq_disable(); early_boot_irqs_disabled.store(true)`.
6. `arch::boot_cpu_init()`; `page_address_init()`.
7. `pr_notice("{}", linux_banner!())`.
8. `let cmdline = arch::setup_arch(); mm_core_init_early(); jump_label_init(); static_call_init(); early_security_init()`.
9. `setup_boot_config()`; `setup_command_line(cmdline)`.
10. `setup_nr_cpu_ids(); setup_per_cpu_areas(); arch::smp_prepare_boot_cpu(); early_numa_node_init(); boot_cpu_hotplug_init()`.
11. `print_kernel_cmdline(saved_command_line)`.
12. `parse_early_param()`.
13. `let after_dashes = parse_args("Booting kernel", static_command_line, &__start___param[..], -1, -1, None, &unknown_bootoption)`.
14. `print_unknown_bootoptions()`.
15. if `after_dashes.is_some()`: `parse_args("Setting init args", after_dashes, &[], -1, -1, None, &set_init_arg)`.
16. if `extra_init_args.is_some()`: `parse_args("Setting extra init args", extra_init_args, ...)`.
17. `random_init_early(cmdline); setup_log_buf(0); vfs_caches_init_early(); sort_main_extable(); arch::trap_init(); mm_core_init(); maple_tree_init(); arch::poking_init(); ftrace_init(); early_trace_init()`.
18. `sched_init()`.
19. `arch::WARN_irqs_enabled(); arch::local_irq_disable(); radix_tree_init(); housekeeping_init(); workqueue_init_early(); rcu_init(); kvfree_rcu_init(); trace_init()`.
20. if `initcall_debug`: `initcall_debug_enable()`.
21. `context_tracking_init(); arch::early_irq_init(); arch::init_IRQ(); tick_init(); rcu_init_nohz(); timers_init(); srcu_init(); hrtimers_init(); softirq_init(); vdso_setup_data_pages(); timekeeping_init(); time_init(); random_init(); kfence_init(); arch::boot_init_stack_canary(); perf_event_init(); profile_init(); call_function_init()`.
22. `early_boot_irqs_disabled.store(false); arch::local_irq_enable()`.
23. `kmem_cache_init_late(); console_init()`.
24. if `panic_later.is_some()`: `panic!("Too many boot {} vars at \`{}\`", panic_later, panic_param)`.
25. `lockdep_init(); locking_selftest()`.
26. CONFIG_BLK_DEV_INITRD: validate `initrd_start` vs. `min_low_pfn`; clear if overwritten.
27. `setup_per_cpu_pageset(); numa_policy_init(); acpi_early_init()`.
28. if `late_time_init.is_some()`: `late_time_init()`.
29. `sched_clock_init(); calibrate_delay(); arch::arch_cpu_finalize_init()`.
30. `pid_idr_init(); anon_vma_init(); thread_stack_cache_init(); cred_init(); fork_init(); proc_caches_init(); uts_ns_init(); time_ns_init(); key_init(); security_init(); dbg_late_init(); net_ns_init(); vfs_caches_init(); pagecache_init(); signals_init(); seq_file_init(); proc_root_init(); nsfs_init(); pidfs_init()`.
31. `cpuset_init(); mem_cgroup_init(); cgroup_init(); taskstats_init_early(); delayacct_init(); acpi_subsystem_init(); arch::arch_post_acpi_subsys_init(); kcsan_init()`.
32. `rest_init()` — `-> !`.

`Init::rest_init() -> !`:
1. `rcu_scheduler_starting()`.
2. `let pid = user_mode_thread(Init::kernel_init, None, CLONE_FS)`.
3. `let tsk = find_task_by_pid_ns(pid, &INIT_PID_NS); tsk.flags |= PF_NO_SETAFFINITY; set_cpus_allowed_ptr(tsk, cpumask_of(smp_processor_id()))`.
4. `numa_default_policy()`.
5. `let pid2 = kernel_thread(kthreadd, None, None, CLONE_FS | CLONE_FILES); KTHREADD_TASK = find_task_by_pid_ns(pid2, &INIT_PID_NS)`.
6. `system_state.store(SYSTEM_SCHEDULING)`.
7. `kthreadd_done.complete()`.
8. `schedule_preempt_disabled()`.
9. `cpu_startup_entry(CPUHP_ONLINE)` — `-> !`.

`Init::kernel_init(_) -> i32`:
1. `kthreadd_done.wait()`.
2. `kernel_init_freeable()`.
3. `async_synchronize_full()`.
4. `system_state.store(SYSTEM_FREEING_INITMEM)`.
5. `kprobe_free_init_mem(); ftrace_free_init_mem(); kgdb_free_init_mem(); exit_boot_config(); free_initmem(); mark_readonly()`.
6. `pti_finalize()`.
7. `system_state.store(SYSTEM_RUNNING)`.
8. `numa_default_policy(); rcu_end_inkernel_boot(); do_sysctl_args()`.
9. if `ramdisk_execute_command.is_some()`: `if run_init_process(ramdisk_execute_command) == 0: return 0; else pr_err`.
10. if `execute_command.is_some()`: `let r = run_init_process(execute_command); if r == 0: return 0; panic!("Requested init {} failed (error {}).", execute_command, r)`.
11. if `CONFIG_DEFAULT_INIT[0] != 0`: `let r = run_init_process(CONFIG_DEFAULT_INIT); if r == 0: return 0`.
12. For `path` in `["/sbin/init", "/etc/init", "/bin/init", "/bin/sh"]`: if `try_to_run_init_process(path) == 0: return 0`.
13. `panic!("No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance.")`.

`Init::kernel_init_freeable()`:
1. `gfp_allowed_mask = __GFP_BITS_MASK`.
2. `set_mems_allowed(node_states[N_MEMORY])`.
3. `cad_pid = get_pid(task_pid(current))`.
4. `smp_prepare_cpus(setup_max_cpus); workqueue_init(); init_mm_internals()`.
5. `do_pre_smp_initcalls()`.
6. `lockup_detector_init(); smp_init(); sched_init_smp(); workqueue_init_topology(); async_init(); padata_init(); page_alloc_init_late()`.
7. `do_basic_setup()`.
8. `kunit_run_all_tests()`.
9. `wait_for_initramfs()`.
10. `console_on_rootfs()`.
11. `let access = init_eaccess(ramdisk_execute_command); if access != 0: if ramdisk_execute_command_set: pr_warn("check access for rdinit=%s failed: %i, ignoring", ...); ramdisk_execute_command = None; prepare_namespace()`.
12. `integrity_load_keys()`.

`Init::do_basic_setup()`:
1. `cpuset_init_smp()`.
2. `ksysfs_init()`.
3. `driver_init()`.
4. `init_irq_proc()`.
5. `do_ctors()` (CONFIG_CONSTRUCTORS && !CONFIG_UML).
6. `do_initcalls()`.

`Init::do_initcalls()`:
1. `let len = saved_command_line_len + 1`.
2. `let cmdline = kzalloc(len, GFP_KERNEL).ok_or_else(|| panic!("{}: Failed to allocate {} bytes", "do_initcalls", len))`.
3. for `level in 0..8`:
   - `cmdline.copy_from_slice(saved_command_line)`.
   - `do_initcall_level(level, &mut cmdline)`.
4. `kfree(cmdline)`.

`Init::do_initcall_level(level, cmdline)`:
1. `parse_args(INITCALL_LEVELS[level], cmdline, &__start___param[..], level as i16, level as i16, None, &ignore_unknown_bootoption)`.
2. `do_trace_initcall_level(INITCALL_LEVELS[level])`.
3. for `fn in initcall_levels[level]..initcall_levels[level + 1]`: `do_one_initcall(initcall_from_entry(fn))`.

`Init::do_one_initcall(fn) -> i32`:
1. `let count = preempt_count()`.
2. if `initcall_blacklisted(fn)`: return `-EPERM`.
3. `do_trace_initcall_start(fn)`.
4. `let ret = fn()`.
5. `do_trace_initcall_finish(fn, ret)`.
6. `let mut msgbuf = [0u8; 64]`.
7. if `preempt_count() != count`: `msgbuf = "preemption imbalance "; preempt_count_set(count)`.
8. if `irqs_disabled()`: `strlcat(&mut msgbuf, "disabled interrupts "); local_irq_enable()`.
9. `WARN!(msgbuf[0] != 0, "initcall {:p} returned with {}", fn, msgbuf)`.
10. `add_latent_entropy()`.
11. return `ret`.

`Init::run_init_process(name) -> i32`:
1. `argv_init[0] = name`.
2. `pr_info("Run {} as init process", name)`.
3. `pr_debug` argv + envp.
4. return `kernel_execve(name, &argv_init, &envp_init)`.

`Init::try_to_run_init_process(name) -> i32`:
1. `let ret = run_init_process(name)`.
2. if `ret != 0 && ret != -ENOENT`: `pr_err("Starting init: {} exists but couldn't execute it (error {})", name, ret)`.
3. return `ret`.

`Init::setup_command_line(cmdline)`:
- See REQ-15 step-by-step.

`Init::setup_boot_config()`:
- See REQ-16 step-by-step.

`Init::mark_readonly()`:
1. if `CONFIG_STRICT_KERNEL_RWX && rodata_enabled`:
   - `flush_module_init_free_work()`.
   - `jump_label_init_ro()`.
   - `mark_rodata_ro()`.
   - `debug_checkwx()`.
   - `rodata_test()`.
2. else-if `CONFIG_STRICT_KERNEL_RWX`: `pr_info("Kernel memory protection disabled.")`.
3. else-if `CONFIG_ARCH_HAS_STRICT_KERNEL_RWX`: `pr_warn("Kernel memory protection not selected by kernel config.")`.
4. else: `pr_warn("This architecture does not have kernel memory protection.")`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `start_kernel_irq_disabled_at_entry` | INVARIANT | per-`start_kernel`: `early_boot_irqs_disabled == true` and `arch::irqs_disabled() == true` after step 6 until step 51. |
| `parse_args_terminator` | INVARIANT | per-`parse_args("Booting kernel", ...)`: returns either `IS_ERR`, `NULL`, or a pointer into `static_command_line` (the `after_dashes` slice). |
| `initcall_level_index_bounds` | INVARIANT | per-`do_initcall_level`: `level < 8` and `initcall_levels[level + 1] >= initcall_levels[level]`. |
| `do_one_initcall_balanced_preempt` | INVARIANT | per-`do_one_initcall`: `preempt_count()` is restored even on initcall imbalance. |
| `do_one_initcall_balanced_irq` | INVARIANT | per-`do_one_initcall`: IRQs are enabled on exit. |
| `kernel_init_search_order` | INVARIANT | per-`kernel_init`: tries `ramdisk_execute_command`, then `execute_command`, then `CONFIG_DEFAULT_INIT`, then `/sbin/init`/`/etc/init`/`/bin/init`/`/bin/sh` — exactly in that order. |
| `init_path_panic_on_explicit_failure` | INVARIANT | per-`kernel_init`: if `execute_command.is_some()` and `run_init_process` non-zero, panic — never fall through. |
| `system_state_monotonic_to_running` | INVARIANT | per-boot: `system_state` only advances along BOOTING → SCHEDULING → FREEING_INITMEM → RUNNING during boot path. |
| `cmdline_buffers_no_overlap` | INVARIANT | per-`setup_command_line`: `saved_command_line` and `static_command_line` are distinct memblock allocations. |

### Layer 2: TLA+

`init/start-kernel.tla`:
- Per-`start_kernel` step transitions modeled as a sequence; each step depends only on prior-step outputs.
- Per-`rest_init` spawns PID 1 then PID 2; completion of `kthreadd_done` precedes `kernel_init`'s wakeup.
- Properties:
  - `safety_pid1_before_pid2_wakeup` — per-`rest_init`: PID 1 created before PID 2.
  - `safety_kthreadd_done_completed_before_kernel_init_freeable` — per-`kernel_init`: waits on `kthreadd_done`.
  - `safety_initcall_levels_monotone` — per-`do_initcalls`: levels 0..7 dispatched in order.
  - `safety_irqs_disabled_until_late` — per-`start_kernel`: IRQs disabled from step 6 to step 51.
  - `safety_system_state_monotone` — per-boot: never regresses.
  - `liveness_kernel_init_reaches_exec_or_panic` — per-`kernel_init`: terminates by execve or panic.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Init::start_kernel` post: `system_state == SYSTEM_SCHEDULING ∨ panic` | `Init::start_kernel` |
| `Init::rest_init` post: PID 1 + PID 2 spawned; idle entered | `Init::rest_init` |
| `Init::kernel_init` post: `execve` succeeded ∨ panic | `Init::kernel_init` |
| `Init::do_initcalls` post: all 8 levels visited | `Init::do_initcalls` |
| `Init::do_one_initcall` post: preempt-count + IRQ state restored | `Init::do_one_initcall` |
| `Init::run_init_process` post: `argv_init[0] == name`; returns `kernel_execve` rv | `Init::run_init_process` |
| `Init::setup_command_line` post: `saved_command_line` ends in `\0` and contains `extra_command_line` prefix + `boot_command_line` + (` -- ` + `extra_init_args`)? | `Init::setup_command_line` |
| `Init::mark_readonly` post: rodata is RO when `CONFIG_STRICT_KERNEL_RWX && rodata_enabled` | `Init::mark_readonly` |

### Layer 4: Verus/Creusot functional

`Per-boot: start_kernel → rest_init → kernel_init → kernel_init_freeable → do_basic_setup → do_initcalls → wait_for_initramfs → run_init_process` semantic equivalence: per-Documentation/admin-guide/init.rst, per-Documentation/admin-guide/kernel-parameters.txt, per-Documentation/core-api/kernel-api.rst (initcall ordering).

## Hardening

(Inherits row-1 features from `init/00-overview.md` § Hardening.)

Boot-driver reinforcement:

- **Per-`early_boot_irqs_disabled` flag** — defense against per-IRQ-handler-runs-too-early bug.
- **Per-`set_task_stack_end_magic(&init_task)` first** — defense against per-idle-task stack overflow undetected.
- **Per-`local_irq_disable` at entry** — defense against per-interrupt-during-fragile-init.
- **Per-`memblock_alloc_or_panic` for cmdline buffers** — defense against per-silent-truncation of `/proc/cmdline`.
- **Per-`panic_later` deferred panic** — defense against per-too-many-init-args before console is up.
- **Per-`initcall_blacklist` requires CONFIG_KALLSYMS** — defense against per-blind-name-match of static functions.
- **Per-`do_one_initcall` WARN on preempt/IRQ imbalance** — defense against per-initcall leaving kernel in inconsistent state.
- **Per-explicit-`init=`-failure panics (no fallback)** — defense against per-`init=/malicious` silently falling through to `/bin/sh`.
- **Per-`run_init_process` fallback chain documented and finite** — defense against per-unbounded init search.
- **Per-`mark_readonly` after `free_initmem`** — defense against per-stale-init-page RW remaining.
- **Per-`flush_module_init_free_work` before `mark_rodata_ro`** — defense against per-W+X transient on module unload.
- **Per-`rodata_test()` runs in `mark_readonly`** — defense against per-RO-not-actually-enforced.
- **Per-`pti_finalize` after `free_initmem`** — defense against per-PTI page-table referencing freed pages.
- **Per-bootconfig `xbc_calc_checksum` validation** — defense against per-corrupt bootconfig blob.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-checked copy on the `boot_command_line` parse path and on the `envp_init[]` / `argv_init[]` arrays handed to `run_init_process`.
- **PAX_KERNEXEC** — `mark_readonly()` is the canonical Rookery hook for write-protecting `.rodata`, `.text`, and the initcall tables; called after `free_initmem` and before `pti_finalize`.
- **PAX_RANDKSTACK** — armed inside `arch_call_rest_init` so the first user-mode `kernel_init` thread inherits stack-offset randomization from boot zero.
- **PAX_REFCOUNT** — saturating wraparound trap on `init_task.usage`, on `system_state` transitions, and on every `__initcall_*` invocation count when audit is enabled.
- **PAX_MEMORY_SANITIZE** — `free_initmem` and `free_initrd_mem` use `POISON_FREE_INITMEM`; `mark_readonly` rejects any RW alias to freed init pages.
- **PAX_UDEREF** — enabled from `setup_arch` onward; `start_kernel` traps any user-pointer deref before `userspace_init` completes.
- **PAX_RAP / kCFI** — forward-edge CFI on `initcall_entry_t` dispatch (`do_one_initcall` indirect call) and on the `init_filename`-resolved `run_init_process` exec hand-off.
- **GRKERNSEC_HIDESYM** — strips initcall and start-kernel symbols from `/proc/kallsyms`; combined with `kptr_restrict=2` in early-boot sysctl seed.
- **GRKERNSEC_DMESG** — restricts the `Booting Linux on physical CPU ...` and initcall-trace lines to CAP_SYSLOG once `setup_log_buf` has handed off.
- **GRKERNSEC_KERN_LOCKDOWN at start_kernel** — lockdown level is decided in `start_kernel` (after `lockdown_init` and before `rest_init`); confidentiality vs. integrity tier is fixed before any user code runs, defends against per-late-policy lockdown bypass.
- **Initcall sequence integrity** — `__initcall_start..__initcall_end` is checksummed (and on `CONFIG_RANDOMIZE_INITCALLS` the seed is recorded) before `do_initcalls`; mismatch ⟹ panic.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `setup_arch` — per-arch; owned by `arch/x86/kernel-platform.md`.
- `boot_cpu_init` internals — per-arch / per-cpu mask; cross-ref `kernel/cpu.md`.
- `mm_core_init_early` / `mm_core_init` / `build_all_zonelists` / `page_alloc_init` (mm bring-up) — covered in `mm/page-allocator.md`.
- `sched_init`, `sched_init_smp` — covered in `kernel/sched/00-overview.md`.
- `rcu_init`, `srcu_init`, `rcu_init_nohz` — covered in `kernel/rcu/00-overview.md`.
- `vfs_caches_init_early` / `vfs_caches_init` / `signals_init` / `proc_root_init` — covered in their respective subsystem docs.
- `prepare_namespace` and `do_mounts*` — covered in `init/rootfs-mount.md`.
- `populate_rootfs` and the initramfs unpacker — covered in `init/initramfs.md`.
- `kernel_execve` / binfmt — covered in `fs/exec.md`.
- `parse_args` parser core — covered in `kernel/params.md`.
- `__setup` / `early_param` / `core_param` linker-section infrastructure — covered in `kernel/params.md`.
- BogoMIPS algorithm — covered in `init/calibrate.md` (folded if not separate).
- Implementation code.
