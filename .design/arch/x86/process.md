# Tier-3: arch/x86/kernel/process.c — x86 generic task lifecycle and idle

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: arch/x86/00-overview.md
upstream-paths:
  - arch/x86/kernel/process.c (~1087 lines)
  - arch/x86/include/asm/processor.h (thread_struct, tss_struct)
  - arch/x86/include/asm/switch_to.h
  - arch/x86/include/asm/fpu/sched.h (fpu_clone, fpu__drop, switch_fpu)
  - arch/x86/include/asm/io_bitmap.h (io_bitmap_share, io_bitmap_exit, tss_update_io_bitmap)
  - arch/x86/include/asm/spec-ctrl.h (TIF_SSBD, TIF_SPEC_IB, TIF_SPEC_FORCE_UPDATE)
  - arch/x86/include/asm/mwait.h (__monitor, __sti_mwait, CPUID_LEAF_MWAIT)
  - arch/x86/include/asm/prctl.h (ARCH_GET/SET_FS/GS/CPUID/XCOMP/SHSTK)
  - include/linux/sched/idle.h (cpu_idle_loop, cpuidle_idle_call)
  - include/linux/entry-common.h (syscall_exit_to_user_mode)
-->

## Summary

`arch/x86/kernel/process.c` implements the **x86 architecture-generic task lifecycle and idle loop**, shared between 32-bit and 64-bit builds. It covers four orthogonal concerns:

1. **Task-struct lifecycle:** `arch_dup_task_struct` (deep-copy + lazy-FPU state), `copy_thread` (fork/clone child setup: stack frame, segment registers, FS/GS bases, FPU clone, shadow stack, TLS, I/O bitmap inheritance), `exit_thread` (release I/O bitmap, vm86, shadow stack, drop FPU), `flush_thread` (exec-side reset of HW breakpoints, TLS array, FPU, PKRU).
2. **Idle loop:** `arch_cpu_idle` (static-call dispatch into one of `default_idle` = HLT, `mwait_idle` = MONITOR/MWAIT, `tdx_halt` = TDX guest, or POLL), `arch_cpu_idle_enter` (TSC verify + NMI touch), `arch_cpu_idle_dead` (CPU-hotunplug → play_dead), `select_idle_routine` (boot-time MWAIT-vs-HALT probe via `prefer_mwait_c1_over_halt`), `stop_this_cpu` (panic/reboot wbinvd + native_halt loop), `idle_setup` early-param parser for `idle=poll|halt|nomwait`.
3. **Context-switch extras (`__switch_to_xtra`):** TIF-bit propagation for I/O bitmap invalidation, user-return notifiers, BLOCKSTEP (DEBUGCTL.BTF), NOTSC (CR4.TSD), NOCPUID (CPUID-fault MSR), and the speculation-control MSRs (SSBD, STIBP, IBPB) via `__speculation_ctrl_update` / `speculation_ctrl_update_current`.
4. **arch-prctl ABI dispatch:** `SYSCALL_DEFINE2(arch_prctl, option, arg2)` — handles `ARCH_GET/SET_CPUID` and `ARCH_GET/REQ_XCOMP_*` here; delegates FS/GS/SHSTK/LAM/VDSO to `do_arch_prctl_64` in `process_64.c`.

Critical for: fork/exec/exit ABI correctness, idle power consumption, secure context switches under Spectre-class CPUs, ptrace/sandbox-visible TIF state.

This Tier-3 covers `arch/x86/kernel/process.c` (~1087 lines). x86_64-specific `__switch_to` body, FSGS swap, debug registers, ARCH_SET_FS/GS handlers are covered in `arch/x86/process-64.md`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `cpu_tss_rw` (per-CPU `struct tss_struct`) | per-CPU TSS, IO bitmap base | `cpu_tss_rw` (PerCpu) |
| `__tss_limit_invalid` | per-CPU lazy-relimit flag | `tss_limit_invalid` (PerCpu) |
| `cache_state_incoherent` | per-CPU wbinvd-needed flag (SME/TDX) | `cache_state_incoherent` (PerCpu) |
| `arch_dup_task_struct()` | per-fork: deep-copy task_struct | `arch::process::arch_dup_task_struct` |
| `arch_release_task_struct()` | per-free: dynamic-XSAVE fpstate_free | `arch::process::arch_release_task_struct` |
| `exit_thread()` | per-exit: io_bitmap_exit, vm86, shstk, fpu drop | `arch::process::exit_thread` |
| `copy_thread()` | per-fork: child frame + FS/GS/TLS/FPU/SHSTK | `arch::process::copy_thread` |
| `flush_thread()` | per-exec: ptrace HW bp, TLS array, FPU, PKRU | `arch::process::flush_thread` |
| `ret_from_fork()` | per-kernel-thread-trampoline → user | `arch::process::ret_from_fork` |
| `set_new_tls()` | per-CLONE_SETTLS: 32/64 dispatch | `arch::process::set_new_tls` |
| `arch_setup_new_exec()` | per-exec: re-enable CPUID, clear TIF_SSBD-noexec | `arch::process::arch_setup_new_exec` |
| `__switch_to_xtra()` | per-switch: TIF-diff side effects | `arch::process::switch_to_xtra` |
| `switch_to_bitmap()` | per-switch: invalidate stale IO bitmap | `arch::process::switch_to_bitmap` |
| `native_tss_update_io_bitmap()` | pre-userret: TSS IO-bitmap copy | `arch::process::tss_update_io_bitmap` |
| `default_idle()` | per-idle: HLT | `arch::process::default_idle` |
| `mwait_idle()` | per-idle: MONITOR/MWAIT | `arch::process::mwait_idle` |
| `arch_cpu_idle()` | per-idle: static-call dispatch | `arch::process::arch_cpu_idle` |
| `arch_cpu_idle_enter()` | per-idle-enter: TSC verify, NMI touch | `arch::process::arch_cpu_idle_enter` |
| `arch_cpu_idle_dead()` | per-CPU-hotunplug | `arch::process::arch_cpu_idle_dead` |
| `stop_this_cpu()` | per-shutdown/panic stop | `arch::process::stop_this_cpu` |
| `select_idle_routine()` | per-boot: probe MWAIT vs HALT | `arch::process::select_idle_routine` |
| `prefer_mwait_c1_over_halt()` | per-boot: CPUID-MWAIT-leaf probe | `arch::process::prefer_mwait_c1_over_halt` |
| `idle_setup()` | per-boot: `idle=` early-param | `arch::process::idle_setup` |
| `amd_e400_c1e_apic_setup()` | per-CPU: AMD E400 broadcast tick | `arch::process::amd_e400_c1e_apic_setup` |
| `arch_post_acpi_subsys_init()` | per-boot: AMD-E400 erratum detect | `arch::process::arch_post_acpi_subsys_init` |
| `speculation_ctrl_update()` | per-prctl/seccomp: MSR refresh | `arch::process::speculation_ctrl_update` |
| `speculation_ctrl_update_current()` | per-current: MSR refresh | `arch::process::speculation_ctrl_update_current` |
| `__speculation_ctrl_update()` | per-switch: SSBD/STIBP MSR update | `arch::process::speculation_ctrl_update_inner` |
| `disable_TSC` / `enable_TSC` | per-prctl: CR4.TSD toggle + TIF_NOTSC | `arch::process::set_tsc_mode` |
| `get_tsc_mode` / `set_tsc_mode` | per-PR_GET/SET_TSC | `arch::process::tsc_mode_*` |
| `disable_cpuid` / `enable_cpuid` | per-prctl: CPUID-fault toggle | `arch::process::set_cpuid_mode` |
| `arch_align_stack()` | per-exec: ASLR-jitter SP | `arch::process::arch_align_stack` |
| `arch_randomize_brk()` | per-exec: ASLR brk | `arch::process::arch_randomize_brk` |
| `__get_wchan()` | /proc/<pid>/wchan via unwinder | `arch::process::get_wchan` |
| `SYSCALL_DEFINE2(arch_prctl, ...)` | per-prctl: x86 ABI dispatch | `arch::process::sys_arch_prctl` |
| `boot_option_idle_override` | per-boot: idle policy enum | `BOOT_OPTION_IDLE_OVERRIDE` |
| `x86_idle` (static_call) | per-idle: indirect dispatch | `X86_IDLE` (static dispatch) |

## Compatibility contract

REQ-1: per-CPU `cpu_tss_rw` (`struct tss_struct`) initialization:
- `.x86_tss.sp0 = (1UL << (BITS_PER_LONG-1)) + 1` — poisoned; init task runs ring 0 only.
- `#ifdef CONFIG_X86_32`: `.sp1 = TOP_OF_INIT_STACK`, `.ss0 = __KERNEL_DS`, `.ss1 = __KERNEL_CS`.
- `.io_bitmap_base = IO_BITMAP_OFFSET_INVALID` — bitmap disabled until first IOPL/ioperm.
- Layout: `DEFINE_PER_CPU_PAGE_ALIGNED` — cacheline-aligned to eliminate ping-pong.

REQ-2: `arch_dup_task_struct(dst, src)`:
- `memcpy_and_pad(dst, arch_task_struct_size, src, sizeof(*dst), 0)` — copies fixed `task_struct` and zero-pads the trailing dynamic FPU region.
- `#ifdef CONFIG_VM86`: `dst->thread.vm86 = NULL` — vm86 state is not inherited.
- Returns 0.

REQ-3: `arch_release_task_struct(tsk)` (x86_64 only):
- If `fpu_state_size_dynamic()` is true and task is neither `PF_KTHREAD` nor `PF_USER_WORKER`: `fpstate_free(x86_task_fpu(tsk))` — releases dynamic AMX XSAVE allocation.

REQ-4: `exit_thread(tsk)`:
- If `TIF_IO_BITMAP`: `io_bitmap_exit(tsk)`.
- `free_vm86(&tsk->thread)`.
- `shstk_free(tsk)`.
- `fpu__drop(tsk)`.

REQ-5: `copy_thread(p, args)` — fork/clone child setup:
- Compute `childregs = task_pt_regs(p)` (top of child kernel stack).
- `fork_frame = container_of(childregs, struct fork_frame, regs)`.
- `frame = &fork_frame->frame` (inactive_task_frame).
- `frame->bp = encode_frame_pointer(childregs)`; `frame->ret_addr = (unsigned long) ret_from_fork_asm`.
- `p->thread.sp = (unsigned long) fork_frame`.
- `p->thread.io_bitmap = NULL`; `clear_tsk_thread_flag(p, TIF_IO_BITMAP)`.
- `p->thread.iopl_warn = 0`; `memset(p->thread.ptrace_bps, 0, sizeof(p->thread.ptrace_bps))`.
- **64-bit:**
  - `current_save_fsgs()` — snapshot current FS/GS bases.
  - Inherit `fsindex`/`fsbase`/`gsindex`/`gsbase` from current.
  - `savesegment(es, p->thread.es)`; `savesegment(ds, p->thread.ds)`.
  - If `p->mm` and `(clone_flags & (CLONE_VM | CLONE_VFORK)) == CLONE_VM`: `set_bit(MM_CONTEXT_LOCK_LAM, &p->mm->context.flags)` — LAM bit-width is now locked per multithreaded mm.
- **32-bit:**
  - `p->thread.sp0 = (unsigned long)(childregs + 1)`.
  - `savesegment(gs, p->thread.gs)`.
  - `frame->flags = X86_EFLAGS_FIXED` — clears IF/AC.
- `new_ssp = shstk_alloc_thread_stack(p, clone_flags, args->stack_size)`; if `IS_ERR_VALUE(new_ssp)`: return `PTR_ERR((void *)new_ssp)`.
- `fpu_clone(p, clone_flags, args->fn, new_ssp)` — propagates `TIF_NEED_FPU_LOAD` to child; child wakes with fresh xfpu state.
- **Kernel-thread branch** (`PF_KTHREAD`):
  - `p->thread.pkru = pkru_get_init_value()`.
  - `memset(childregs, 0, sizeof(struct pt_regs))`.
  - `kthread_frame_init(frame, args->fn, args->fn_arg)`.
  - return 0.
- **User-thread branch:**
  - `p->thread.pkru = read_pkru()`.
  - `frame->bx = 0`; `*childregs = *current_pt_regs()`; `childregs->ax = 0` (fork returns 0 in child).
  - If `sp != 0`: `childregs->sp = sp` (clone-supplied stack).
  - **user-worker branch** (`args->fn != NULL` even for non-PF_KTHREAD): `childregs->sp = 0`; `childregs->ip = 0`; `kthread_frame_init(frame, args->fn, args->fn_arg)`; return 0.
  - If `clone_flags & CLONE_SETTLS`: `ret = set_new_tls(p, tls)`.
  - If `!ret` and `TIF_IO_BITMAP` on current: `io_bitmap_share(p)`.
  - return `ret`.

REQ-6: `set_new_tls(p, tls)`:
- If `in_ia32_syscall()`: `do_set_thread_area(p, -1, (struct user_desc __user *)tls, 0)`.
- Else: `do_set_thread_area_64(p, ARCH_SET_FS, tls)`.

REQ-7: `ret_from_fork(prev, regs, fn, fn_arg)` — C trampoline called from `ret_from_fork_asm`:
- `schedule_tail(prev)` — finalize prior task's switch-out.
- If `unlikely(fn)`: `fn(fn_arg)`; `regs->ax = 0` (kernel-thread post-`kernel_execve` returns 0 in `ax`).
- `syscall_exit_to_user_mode(regs)` — entry-common path to userspace.

REQ-8: `flush_thread()` — exec-time reset:
- `flush_ptrace_hw_breakpoint(current)`.
- `memset(current->thread.tls_array, 0, sizeof(current->thread.tls_array))`.
- `fpu_flush_thread()` — re-init FPU + clear `TIF_NEED_FPU_LOAD` semantics.
- `pkru_flush_thread()` — load PKRU default.

REQ-9: `arch_setup_new_exec()`:
- If `TIF_NOCPUID` set: `enable_cpuid()` — exec clears CPUID-fault.
- If `TIF_SSBD` set ∧ `task_spec_ssb_noexec(current)`: clear `TIF_SSBD`, `task_clear_spec_ssb_disable`, `task_clear_spec_ssb_noexec`, `speculation_ctrl_update(read_thread_flags())`.
- `mm_reset_untag_mask(current->mm)` — LAM untag mask reset for new ELF.

REQ-10: `__switch_to_xtra(prev_p, next_p)` — invoked from `__switch_to` for non-fast-path TIF-diff work:
- `tifn = read_task_thread_flags(next_p)`; `tifp = read_task_thread_flags(prev_p)`.
- `switch_to_bitmap(tifp)`:
  - If `tifp & _TIF_IO_BITMAP`: `tss_invalidate_io_bitmap()` — incoming task handles bitmap on userret.
- `propagate_user_return_notify(prev_p, next_p)`.
- If `(tifp & _TIF_BLOCKSTEP || tifn & _TIF_BLOCKSTEP) ∧ arch_has_block_step()`: RMW `MSR_IA32_DEBUGCTLMSR` BTF bit per `tifn` BLOCKSTEP.
- If `(tifp ^ tifn) & _TIF_NOTSC`: `cr4_toggle_bits_irqsoff(X86_CR4_TSD)`.
- If `(tifp ^ tifn) & _TIF_NOCPUID`: `set_cpuid_faulting(!!(tifn & _TIF_NOCPUID))`.
- Spec-ctrl: if neither side has `_TIF_SPEC_FORCE_UPDATE`: `__speculation_ctrl_update(tifp, tifn)`; else `speculation_ctrl_update_tif(prev_p)`, `tifn = speculation_ctrl_update_tif(next_p)`, `__speculation_ctrl_update(~tifn, tifn)`.

REQ-11: `__speculation_ctrl_update(tifp, tifn)`:
- `lockdep_assert_irqs_disabled()`.
- `tif_diff = tifp ^ tifn`; `msr = x86_spec_ctrl_base`; `updmsr = false`.
- AMD VIRT_SSBD: if diff includes `_TIF_SSBD`: `amd_set_ssb_virt_state(tifn)`.
- AMD LS_CFG_SSBD: if diff includes `_TIF_SSBD`: `amd_set_core_ssb_state(tifn)`.
- Intel/AMD SPEC_CTRL_SSBD or AMD_SSBD: `updmsr |= (tif_diff & _TIF_SSBD) != 0`; `msr |= ssbd_tif_to_spec_ctrl(tifn)`.
- Conditional STIBP (`switch_to_cond_stibp`): `updmsr |= (tif_diff & _TIF_SPEC_IB) != 0`; `msr |= stibp_tif_to_spec_ctrl(tifn)`.
- If `updmsr`: `update_spec_ctrl_cond(msr)`.

REQ-12: `default_idle()` (HLT):
- `raw_safe_halt()` — `STI; HLT` atomically.
- `raw_local_irq_disable()` after wake to match arch-generic contract.

REQ-13: `mwait_idle()` (MONITOR/MWAIT, C1 substate):
- If `need_resched()`: return early.
- `x86_idle_clear_cpu_buffers()` — VERW for MDS mitigation.
- `current_set_polling_and_test()` — if no resched pending:
  - `addr = &current_thread_info()->flags`.
  - `alternative_input("", "clflush (%[addr])", X86_BUG_CLFLUSH_MONITOR, ...)` — pre-MONITOR clflush erratum workaround.
  - `__monitor(addr, 0, 0)`.
  - If `need_resched()`: goto out.
  - `__sti_mwait(0, 0)` — STI; MWAIT(0, 0).
  - `raw_local_irq_disable()`.
- out: `__current_clr_polling()`.

REQ-14: `arch_cpu_idle()`:
- `static_call(x86_idle)()` — dispatches to `default_idle`, `mwait_idle`, `tdx_halt`, or polling per `select_idle_routine`.

REQ-15: `arch_cpu_idle_enter()`:
- `tsc_verify_tsc_adjust(false)` — sanity-check IA32_TSC_ADJUST.
- `local_touch_nmi()` — reset NMI-watchdog counter so an idling CPU is not declared "stuck."

REQ-16: `arch_cpu_idle_dead()`:
- `play_dead()` — CPU-hotplug offline; never returns.

REQ-17: `select_idle_routine()` (boot-time):
- If `boot_option_idle_override == IDLE_POLL`: warn if SMT enabled; return (POLL path uses cpu_idle_poll).
- If `x86_idle_set()` already (e.g. Xen): return.
- If `prefer_mwait_c1_over_halt()`: `static_call_update(x86_idle, mwait_idle)`.
- Else if `cpu_feature_enabled(X86_FEATURE_TDX_GUEST)`: `static_call_update(x86_idle, tdx_halt)`.
- Else: `static_call_update(x86_idle, default_idle)`.

REQ-18: `prefer_mwait_c1_over_halt()` requires:
- `boot_option_idle_override == IDLE_NO_OVERRIDE`.
- `cpu_has(c, X86_FEATURE_MWAIT)`.
- `!boot_cpu_has_bug(X86_BUG_MONITOR) ∧ !boot_cpu_has_bug(X86_BUG_AMD_APIC_C1E)`.
- `cpuid(CPUID_LEAF_MWAIT, ...)`: either `!(ecx & CPUID5_ECX_EXTENSIONS_SUPPORTED)` or `(edx & MWAIT_C1_SUBSTATE_MASK) != 0`.

REQ-19: `stop_this_cpu(dummy)`:
- `local_irq_disable()`.
- `set_cpu_online(cpu, false)`; `disable_local_APIC()`; `mcheck_cpu_clear(c)`.
- If `cache_state_incoherent`: `wbinvd()`.
- `cpumask_clear_cpu(cpu, &cpus_stop_mask)`.
- If `smp_ops.stop_this_cpu`: call it; `BUG()` if returns.
- Loop forever: `native_halt()`.

REQ-20: `idle_setup(str)` early-param:
- `"poll"`: `IDLE_POLL`; `cpu_idle_poll_ctrl(true)`.
- `"halt"`: `IDLE_HALT`.
- `"nomwait"`: `IDLE_NOMWAIT`.
- Else: `-EINVAL`.

REQ-21: `arch_align_stack(sp)`:
- If `!(current->personality & ADDR_NO_RANDOMIZE) ∧ randomize_va_space`: `sp -= get_random_u32_below(8192)`.
- return `sp & ~0xf`.

REQ-22: `arch_randomize_brk(mm)`:
- If `mmap_is_ia32()`: `randomize_page(mm->brk, SZ_32M)`.
- Else: `randomize_page(mm->brk, SZ_1G)`.

REQ-23: `__get_wchan(p)`:
- `try_get_task_stack(p)` — if 0, return 0.
- Walk frames via `unwind_start` / `unwind_next_frame` / `unwind_get_return_address`; first frame outside `in_sched_functions(addr)` wins.
- `put_task_stack(p)`.

REQ-24: `SYSCALL_DEFINE2(arch_prctl, option, arg2)`:
- `ARCH_GET_CPUID`: `get_cpuid_mode()`.
- `ARCH_SET_CPUID`: `set_cpuid_mode(arg2)`.
- `ARCH_GET_XCOMP_SUPP / _PERM / _GUEST_PERM`, `ARCH_REQ_XCOMP_PERM / _GUEST_PERM`: `fpu_xstate_prctl(option, arg2)`.
- Else if `!in_ia32_syscall()`: `do_arch_prctl_64(current, option, arg2)`.
- Else: `-EINVAL`.

REQ-25: `disable_TSC()` / `enable_TSC()`:
- `preempt_disable`.
- `test_and_set_thread_flag(TIF_NOTSC)` (or `test_and_clear`): if flipped, `cr4_set_bits(X86_CR4_TSD)` (or `cr4_clear_bits`).
- `preempt_enable`.

REQ-26: `set_cpuid_faulting(on)`:
- Intel: RMW `MSR_MISC_FEATURES_ENABLES` `CPUID_FAULT` bit via per-CPU shadow `msr_misc_features_shadow`.
- AMD: `msr_set_bit` / `msr_clear_bit` on `MSR_K7_HWCR` bit `MSR_K7_HWCR_CPUID_USER_DIS_BIT`.

REQ-27: `set_cpuid_mode(cpuid_enabled)`:
- Require `boot_cpu_has(X86_FEATURE_CPUID_FAULT)` — else `-ENODEV`.
- Toggle via `enable_cpuid()` / `disable_cpuid()`.

## Acceptance Criteria

- [ ] AC-1: `arch_dup_task_struct`: child's first `arch_task_struct_size` bytes equal parent's; trailing pad bytes are zero; child `thread.vm86 == NULL`.
- [ ] AC-2: `copy_thread` user-thread path: `childregs == *current_pt_regs()` except `ax = 0`; `frame->ret_addr == ret_from_fork_asm`; `thread.sp` points at the fork_frame.
- [ ] AC-3: `copy_thread` PF_KTHREAD path: `childregs` zeroed; `kthread_frame_init` sets `fn`/`fn_arg`; PKRU = init value.
- [ ] AC-4: `copy_thread` 64-bit: `fsbase`/`gsbase`/`fsindex`/`gsindex`/`es`/`ds` inherited from current at fork instant (`current_save_fsgs` ran).
- [ ] AC-5: `copy_thread` shadow-stack: returns `-ENOMEM` (propagated from `shstk_alloc_thread_stack`) when shstk allocation fails; otherwise `fpu_clone` receives a valid `new_ssp`.
- [ ] AC-6: `copy_thread` CLONE_SETTLS: invokes `do_set_thread_area` (32-bit) or `do_set_thread_area_64(ARCH_SET_FS, tls)` (64-bit).
- [ ] AC-7: `copy_thread` TIF_IO_BITMAP on parent: `io_bitmap_share(p)` increments bitmap refcount on child.
- [ ] AC-8: `flush_thread` (exec): TLS array zeroed; HW breakpoints flushed; PKRU = default; FPU re-init.
- [ ] AC-9: `exit_thread` releases I/O bitmap (if TIF_IO_BITMAP), vm86 state, shadow stack, and drops FPU.
- [ ] AC-10: `arch_cpu_idle` invokes the routine selected by `select_idle_routine`; with `idle=poll`, no `arch_cpu_idle` call occurs (POLL handled by cpu_idle_poll).
- [ ] AC-11: `mwait_idle` issues MONITOR(addr=current_thread_info()->flags, 0, 0) then `STI; MWAIT`; aborts on `need_resched()` between MONITOR and MWAIT.
- [ ] AC-12: `prefer_mwait_c1_over_halt`: returns false when `X86_BUG_MONITOR` or `X86_BUG_AMD_APIC_C1E`, when MWAIT feature absent, or when MWAIT-leaf reports no C1 substate.
- [ ] AC-13: `__switch_to_xtra` SSBD diff: `MSR_SPEC_CTRL` written iff TIF_SSBD changed (and CPU has SPEC_CTRL_SSBD/AMD_SSBD).
- [ ] AC-14: `stop_this_cpu`: sets CPU offline; if `cache_state_incoherent` true, executes WBINVD; spins in `native_halt()`.
- [ ] AC-15: `arch_prctl` IA32 path rejects 64-bit-only ops (`ARCH_GET/SET_FS/GS`) with `-EINVAL`.

## Architecture

```
struct ThreadStruct (subset relevant to process.c) {
  sp:        u64,                    // kernel stack pointer at switch
  sp0:       u64,                    // 32-bit only
  fsindex:   u16,                    // 64-bit
  gsindex:   u16,                    // 64-bit
  fsbase:    u64,                    // 64-bit (linear FS base)
  gsbase:    u64,                    // 64-bit (KERNEL_GS_BASE for inactive)
  es:        u16, ds: u16,           // 64-bit
  gs:        u16,                    // 32-bit
  tls_array: [Desc; GDT_ENTRY_TLS_ENTRIES],
  io_bitmap: Option<*IoBitmap>,
  iopl_warn: u8,
  iopl_emul: u8,
  ptrace_bps:[*PerfEvent; HBP_NUM],
  pkru:      u32,
  vm86:      Option<*Vm86Struct>,
  /* fpu trailing dynamic state (variable size) */
}
```

`arch::process::arch_dup_task_struct(dst, src) -> i32`:
1. memcpy_and_pad(dst, arch_task_struct_size, src, sizeof(TaskStruct), 0).
2. dst.thread.vm86 = None (cfg vm86).
3. return 0.

`arch::process::copy_thread(p, args) -> Result<(), Errno>`:
1. childregs = task_pt_regs(p).
2. fork_frame = container_of(childregs, ForkFrame, regs).
3. frame = &fork_frame.frame.
4. frame.bp = encode_frame_pointer(childregs).
5. frame.ret_addr = ret_from_fork_asm as u64.
6. p.thread.sp = fork_frame as u64.
7. p.thread.io_bitmap = None; clear TIF_IO_BITMAP; p.thread.iopl_warn = 0; zero ptrace_bps.
8. /* 64-bit */
   - current_save_fsgs().
   - copy fsindex/fsbase/gsindex/gsbase from current.
   - savesegment(es, p.thread.es); savesegment(ds, p.thread.ds).
   - if p.mm && (clone_flags & (CLONE_VM|CLONE_VFORK)) == CLONE_VM: set MM_CONTEXT_LOCK_LAM.
9. /* 32-bit */
   - p.thread.sp0 = (childregs+1) as u64.
   - savesegment(gs, p.thread.gs).
   - frame.flags = X86_EFLAGS_FIXED.
10. new_ssp = shstk_alloc_thread_stack(p, clone_flags, args.stack_size).
11. if new_ssp is err: return Err.
12. fpu_clone(p, clone_flags, args.fn, new_ssp).
13. if PF_KTHREAD:
    - p.thread.pkru = pkru_get_init_value().
    - zero childregs.
    - kthread_frame_init(frame, args.fn, args.fn_arg).
    - return Ok.
14. p.thread.pkru = read_pkru().
15. frame.bx = 0; *childregs = *current_pt_regs(); childregs.ax = 0.
16. if args.stack: childregs.sp = args.stack.
17. if args.fn (user-worker):
    - childregs.sp = 0; childregs.ip = 0.
    - kthread_frame_init(frame, args.fn, args.fn_arg).
    - return Ok.
18. if CLONE_SETTLS: ret = set_new_tls(p, args.tls).
19. if ret == 0 && TIF_IO_BITMAP on current: io_bitmap_share(p).
20. return ret.

`arch::process::__switch_to_xtra(prev, next)`:
1. tifn = read_task_thread_flags(next); tifp = read_task_thread_flags(prev).
2. switch_to_bitmap(tifp).
3. propagate_user_return_notify(prev, next).
4. BLOCKSTEP: if either side BLOCKSTEP and arch_has_block_step(): rdmsrq DEBUGCTL, clear BTF, set BTF from tifn, wrmsrq.
5. NOTSC: if (tifp ^ tifn) & _TIF_NOTSC: cr4_toggle_bits_irqsoff(X86_CR4_TSD).
6. NOCPUID: if (tifp ^ tifn) & _TIF_NOCPUID: set_cpuid_faulting(...).
7. Spec-ctrl: per-FORCE-UPDATE path or fast path.

`arch::process::arch_cpu_idle()`:
1. static_call(x86_idle)().

`arch::process::default_idle()`:
1. raw_safe_halt().            // STI; HLT
2. raw_local_irq_disable().

`arch::process::mwait_idle()`:
1. if need_resched(): return.
2. x86_idle_clear_cpu_buffers().
3. if !current_set_polling_and_test():
   - addr = &current_thread_info().flags.
   - alternative_input clflush (CLFLUSH_MONITOR erratum).
   - __monitor(addr, 0, 0).
   - if need_resched(): goto out.
   - __sti_mwait(0, 0).
   - raw_local_irq_disable().
4. out: __current_clr_polling().

`arch::process::select_idle_routine()`:
1. if IDLE_POLL: warn-on-SMT; return.
2. if x86_idle_set(): return.
3. if prefer_mwait_c1_over_halt(): static_call_update(x86_idle, mwait_idle).
4. else if TDX_GUEST: static_call_update(x86_idle, tdx_halt).
5. else: static_call_update(x86_idle, default_idle).

`arch::process::stop_this_cpu(_dummy)`:
1. local_irq_disable.
2. set_cpu_online(cpu, false); disable_local_APIC; mcheck_cpu_clear.
3. if cache_state_incoherent: wbinvd.
4. cpumask_clear_cpu(cpu, &cpus_stop_mask).
5. if smp_ops.stop_this_cpu: call; BUG! if it returns.
6. loop { native_halt(); }.

`arch::process::sys_arch_prctl(option, arg2) -> i64`:
1. switch option:
   - ARCH_GET_CPUID: get_cpuid_mode().
   - ARCH_SET_CPUID: set_cpuid_mode(arg2).
   - ARCH_*_XCOMP_*: fpu_xstate_prctl(option, arg2).
2. if !in_ia32_syscall(): do_arch_prctl_64(current, option, arg2).
3. else: -EINVAL.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dup_task_pad_zero` | INVARIANT | per-arch_dup_task_struct: bytes [sizeof(TaskStruct), arch_task_struct_size) are zero in dst. |
| `copy_thread_ret_addr_is_fork_asm` | INVARIANT | per-copy_thread: frame.ret_addr == &ret_from_fork_asm. |
| `copy_thread_io_bitmap_cleared` | INVARIANT | per-copy_thread: TIF_IO_BITMAP cleared on child before optional io_bitmap_share. |
| `copy_thread_kthread_regs_zeroed` | INVARIANT | per-copy_thread PF_KTHREAD: childregs is all-zero. |
| `copy_thread_user_ax_is_zero` | INVARIANT | per-copy_thread user path: childregs.ax == 0. |
| `mwait_idle_no_resched_loss` | INVARIANT | per-mwait_idle: between MONITOR and MWAIT, need_resched check honored. |
| `xtra_msr_under_irq_off` | INVARIANT | per-__speculation_ctrl_update: lockdep IRQs-off. |
| `stop_this_cpu_irq_off` | INVARIANT | per-stop_this_cpu: local_irq_disable before any state mutation. |
| `arch_prctl_ia32_rejects_64bit_ops` | INVARIANT | per-sys_arch_prctl: in_ia32_syscall ∧ unknown option ⟹ -EINVAL. |

### Layer 2: TLA+

`arch/x86/process.tla`:
- States the full task lifecycle: ALLOC → DUP → COPY → RUN ↔ SWITCH ↔ IDLE → EXIT → FREE.
- Idle subsystem: CHOOSE x86_idle ∈ {default, mwait, tdx, poll}; LOOP on need_resched.
- Properties:
  - `safety_io_bitmap_invalidated_on_switch_out` — per-switch: if tifp has _TIF_IO_BITMAP, tss_invalidate_io_bitmap fires.
  - `safety_fpu_load_propagated` — per-copy_thread: TIF_NEED_FPU_LOAD on child iff fpu_clone propagated it.
  - `safety_kthread_never_returns_to_user` — per-kthread: childregs.ip == 0; ret_from_fork drops to syscall_exit_to_user_mode only after fn returns successfully.
  - `safety_mwait_addr_is_thread_info_flags` — per-mwait_idle: MONITOR address always &current_thread_info()->flags.
  - `liveness_arch_cpu_idle_terminates` — per-arch_cpu_idle: eventually need_resched or interrupt.
  - `safety_stop_this_cpu_no_progress` — per-stop_this_cpu: post-call, CPU forever halted.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `arch_dup_task_struct` post: dst[0..sizeof(TaskStruct)] == src; dst[sizeof..arch_task_struct_size] == 0; dst.thread.vm86 == None | `arch_dup_task_struct` |
| `copy_thread` post (user): childregs.ax == 0; thread.sp on child's fork_frame; ret_from_fork_asm linked | `copy_thread` |
| `copy_thread` post (kthread): childregs == 0; kthread_frame_init called with fn,fn_arg | `copy_thread` |
| `__switch_to_xtra` post: MSR writes minimal (only on TIF diff or FORCE_UPDATE) | `__switch_to_xtra` |
| `mwait_idle` post: __current_clr_polling executed on all return paths | `mwait_idle` |
| `select_idle_routine` post: static_call(x86_idle) bound exactly once (idempotent across calls) | `select_idle_routine` |
| `stop_this_cpu` post: cpu_online(cpu) == false; cpus_stop_mask bit clear | `stop_this_cpu` |
| `arch_align_stack` post: result ≡ 0 (mod 16); result ≤ original; original − result < 8192 | `arch_align_stack` |

### Layer 4: Verus/Creusot functional

`fork(2) → copy_thread → schedule → __switch_to → __switch_to_xtra → ret_from_fork → syscall_exit_to_user_mode → user`:
- Semantic equivalence to the C path for: child sees `ax=0`, `current_pt_regs` clone, FPU state freshly cloned, FS/GS bases inherited.

`exit(2) → exit_thread → arch_release_task_struct`:
- Semantic equivalence: I/O bitmap freed iff TIF_IO_BITMAP held; shadow stack freed; FPU dropped; dynamic XSAVE freed for non-kthread/non-user-worker.

`idle path: cpu_idle_loop → arch_cpu_idle_enter → arch_cpu_idle (static-call) → arch_cpu_idle_exit → need_resched`:
- Semantic equivalence per `Documentation/admin-guide/kernel-parameters.txt` (`idle=` semantics).

## Hardening

(Inherits arch baseline from `arch/x86/00-overview.md` § Hardening.)

Process-and-idle reinforcement:

- **arch_task_struct_size pad zero (memcpy_and_pad)** — defense against per-fork stale-state leak across kernel versions.
- **TIF_IO_BITMAP cleared on every fork before optional share** — defense against per-fork I/O-port-perm inheritance bug.
- **Shadow-stack allocation failure propagated as Err, not panic** — defense against per-OOM-during-fork crash.
- **`__speculation_ctrl_update` requires IRQs off (lockdep_assert)** — defense against per-MSR-thrash race with NMI.
- **MWAIT only after CLFLUSH-monitor erratum + `X86_BUG_MONITOR`/`X86_BUG_AMD_APIC_C1E` exclusion** — defense against per-stale-MONITOR-line missed-wakeup.
- **MWAIT path always issues `__current_clr_polling` on exit** — defense against per-stale-polling deadlock.
- **`__switch_to_xtra` MSR writes gated on actual diff** — defense against per-context-switch slowdown / unnecessary IBPB.
- **TIF_SPEC_FORCE_UPDATE forces both ~tifn and tifn write** — defense against per-prctl/seccomp stale-SSBD/STIBP MSR.
- **`stop_this_cpu` wbinvd gated on `cache_state_incoherent`** — defense against per-SME/TDX-dirty-cache silent corruption on kexec.
- **`disable_local_APIC` + offline before halt loop** — defense against per-stuck-CPU spurious-interrupt storm.
- **`arch_align_stack` 16-byte alignment + ≤ 8192-byte ASLR jitter** — defense against per-exec stack-layout fingerprint.
- **`arch_randomize_brk` SZ_1G (64-bit) / SZ_32M (ia32)** — defense against per-heap-layout fingerprint.
- **`set_cpuid_faulting` per-vendor (Intel MSR_MISC_FEATURES vs AMD MSR_K7_HWCR)** — defense against per-wrong-MSR-touched-on-wrong-CPU silent failure.
- **AMD E400 broadcast-tick workaround (`amd_e400_c1e_apic_setup`)** — defense against per-AMD-C1E-APIC-stop missed-tick.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `arch/x86/kernel/process_64.c` x86_64 switch_to body / FSGS swap / ARCH_SET_GS prctl (covered in `arch/x86/process-64.md`).
- `arch/x86/kernel/process_32.c` 32-bit switch_to body (covered separately if 32-bit is ever a Rookery target).
- `arch/x86/kernel/fpu/*` FPU state save/restore internals (covered in `arch/x86/fpu.md`).
- `arch/x86/kernel/shstk.c` shadow-stack allocator (covered in `arch/x86/shstk.md`).
- `arch/x86/kernel/io_bitmap.c` bitmap lifetime (covered in `arch/x86/io-bitmap.md`).
- `kernel/fork.c` generic `dup_task_struct` / `copy_process` (covered in `fork.md`).
- `kernel/sched/idle.c` cpu_idle_loop (covered in `sched-idle.md`).
- `drivers/cpuidle/*` deeper C-states (covered separately if expanded).
- Implementation code.
