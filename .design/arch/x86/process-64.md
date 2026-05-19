# Tier-3: arch/x86/kernel/process_64.c — x86_64 process switch and arch-prctl

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: arch/x86/00-overview.md
upstream-paths:
  - arch/x86/kernel/process_64.c (~982 lines)
  - arch/x86/entry/entry_64.S (__switch_to_asm, ret_from_fork_asm)
  - arch/x86/include/asm/switch_to.h (switch_to, switch_to_extra)
  - arch/x86/include/asm/fsgsbase.h (rdfsbase, wrfsbase, rdgsbase, wrgsbase, native_swapgs)
  - arch/x86/include/asm/fpu/sched.h (switch_fpu)
  - arch/x86/include/asm/desc.h (load_TLS, loadsegment, savesegment, load_gs_index)
  - arch/x86/include/asm/prctl.h (ARCH_GET/SET_FS/GS, ARCH_SHSTK_*, ARCH_*_TAGGED_ADDR, ARCH_MAP_VDSO_*)
  - arch/x86/include/asm/proto.h
  - arch/x86/include/uapi/asm/prctl.h
  - include/uapi/linux/elf.h (NT_X86_XSTATE etc.)
-->

## Summary

`arch/x86/kernel/process_64.c` implements the **x86_64-specific half of process switching, debug-state, and the arch_prctl ABI**:

1. **`__switch_to(prev, next)`** — the C body called from `__switch_to_asm` (entry_64.S) after the assembly half has swapped kernel stacks and callee-saved registers. Performs: FPU switch (`switch_fpu`), `save_fsgs(prev)`, GDT TLS load (`load_TLS(next, cpu)`), paravirt `arch_end_context_switch`, DS/ES selector reload, `x86_fsgsbase_load(prev, next)` (FS/GS bases, FSGSBASE-aware), PKRU swap (`x86_pkru_load`), per-CPU `current_task` / `cpu_current_top_of_stack` write, `update_task_stack` (TSS sp0 reload), `switch_to_extra` (debug regs + I/O bitmap + xtra TIF), AMD SYSRET-SS NULL fixup, `resctrl_arch_sched_in`, AMD workload-class reset.
2. **FS/GS base management** — `__rdgsbase_inactive` / `__wrgsbase_inactive` (FRED-aware swapgs+rdgsbase/wrgsbase), `save_fsgs` (FSGSBASE vs legacy `save_base_legacy`), `load_seg_legacy` (no-FSGSBASE pre-baseline path), `current_save_fsgs` (IRQ-off wrapper), `x86_fsgsbase_load` (FSGSBASE fast path: rdfsbase/wrfsbase + `__wrgsbase_inactive`), `x86_fsbase_read_task` / `x86_gsbase_read_task` (ptrace + ARCH_GET_FS/GS path, including LDT lookup via `x86_fsgsbase_read_task`), `x86_fsbase_write_task` / `x86_gsbase_write_task` (refuses current — must use ARCH_SET_FS/GS path).
3. **Thread start (`start_thread`, `start_thread_common`, `compat_start_thread`)** — clears DS/ES, loads `_ds`, sets `regs->{ip,sp,csx,ssx}`, FRED swevent/nmi bits, `EFLAGS = IF | FIXED`.
4. **Personality (`set_personality_64bit`, `set_personality_ia32`)** — TIF_ADDR32, TS_COMPAT, ORIG_RAX = __NR_(x32_)?execve / __NR_ia32_execve, MM_CONTEXT_HAS_VSYSCALL / MM_CONTEXT_UPROBE_IA32.
5. **`__show_regs(regs, mode, lvl)`** — oops/panic register dump including FS_BASE/KERNEL_GS_BASE MSRs, CR0/CR2/CR3/CR4, DR0–DR7 (only if non-default), and PKRU.
6. **`do_arch_prctl_64(task, option, arg2)`** — implements `ARCH_SET_FS`, `ARCH_SET_GS`, `ARCH_GET_FS`, `ARCH_GET_GS`, `ARCH_MAP_VDSO_{32,64,X32}`, `ARCH_GET_UNTAG_MASK` / `ARCH_ENABLE_TAGGED_ADDR` / `ARCH_FORCE_TAGGED_SVA` / `ARCH_GET_MAX_TAG_BITS` (LAM), and `ARCH_SHSTK_*` (delegates to `shstk_prctl`).
7. **`release_thread(dead_task)`** — final WARN if `mm` not torn down.

This Tier-3 covers `arch/x86/kernel/process_64.c` (~982 lines). The generic task lifecycle (`copy_thread`, `exit_thread`, `arch_dup_task_struct`, idle, arch_prctl front-end) is in `arch/x86/process.md`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `__switch_to_asm` (asm) | per-switch: stack/regs swap, RET into __switch_to | `arch::process64::switch_to_asm` (assembly) |
| `__switch_to()` | per-switch: C body | `arch::process64::switch_to` |
| `__show_regs()` | per-oops: register/CR/DR/MSR dump | `arch::process64::show_regs` |
| `release_thread()` | per-free: WARN on residual mm | `arch::process64::release_thread` |
| `__rdgsbase_inactive()` | noinstr: read inactive GS base (SWAPGS or KERNEL_GS_BASE) | `arch::process64::rdgsbase_inactive` |
| `__wrgsbase_inactive()` | noinstr: write inactive GS base | `arch::process64::wrgsbase_inactive` |
| `save_base_legacy()` | per-prev: legacy FS/GS base save | `arch::process64::save_base_legacy` |
| `save_fsgs()` | per-prev: FSGSBASE-aware save | `arch::process64::save_fsgs` |
| `current_save_fsgs()` | per-current: IRQ-off save_fsgs wrapper | `arch::process64::current_save_fsgs` |
| `loadseg()` | per-FS/GS selector load | `arch::process64::loadseg` |
| `load_seg_legacy()` | per-next: legacy FS/GS load | `arch::process64::load_seg_legacy` |
| `x86_pkru_load()` | per-switch: PKRU swap (gated on diff) | `arch::process64::pkru_load` |
| `x86_fsgsbase_load()` | per-switch: FSGSBASE fast path | `arch::process64::fsgsbase_load` |
| `x86_fsgsbase_read_task()` | per-task: GDT TLS or LDT base lookup | `arch::process64::fsgsbase_read_task` |
| `x86_gsbase_read_cpu_inactive()` | per-CPU: read KERNEL_GS_BASE | `arch::process64::gsbase_read_cpu_inactive` |
| `x86_gsbase_write_cpu_inactive()` | per-CPU: write KERNEL_GS_BASE | `arch::process64::gsbase_write_cpu_inactive` |
| `x86_fsbase_read_task()` | per-task: FS base reader | `arch::process64::fsbase_read_task` |
| `x86_gsbase_read_task()` | per-task: GS base reader | `arch::process64::gsbase_read_task` |
| `x86_fsbase_write_task()` | per-task: FS base writer (non-current) | `arch::process64::fsbase_write_task` |
| `x86_gsbase_write_task()` | per-task: GS base writer (non-current) | `arch::process64::gsbase_write_task` |
| `start_thread_common()` | per-exec: regs/seg setup | `arch::process64::start_thread_common` |
| `start_thread()` | per-exec: 64-bit | `arch::process64::start_thread` |
| `compat_start_thread()` | per-exec: x32 / ia32 | `arch::process64::compat_start_thread` |
| `set_personality_64bit()` | per-exec: TIF_ADDR32 clear, ORIG_RAX = __NR_execve | `arch::process64::set_personality_64bit` |
| `set_personality_ia32()` | per-exec: TIF_ADDR32 set, TS_COMPAT / x32 split | `arch::process64::set_personality_ia32` |
| `__set_personality_x32()` | per-exec: x32 subpath | `arch::process64::set_personality_x32` |
| `__set_personality_ia32()` | per-exec: ia32 subpath | `arch::process64::set_personality_ia32_inner` |
| `prctl_map_vdso()` | per-prctl: CHECKPOINT_RESTORE vdso remap | `arch::process64::prctl_map_vdso` |
| `mm_enable_lam()` | per-mm: LAM_U57 CR3 mask + IPI | `arch::process64::mm_enable_lam` |
| `enable_lam_func()` | per-CPU: write_cr3 + tlbstate update | `arch::process64::enable_lam_func` |
| `prctl_enable_tagged_addr()` | per-prctl: LAM enable | `arch::process64::prctl_enable_tagged_addr` |
| `do_arch_prctl_64()` | per-prctl: 64-bit ABI dispatch | `arch::process64::do_arch_prctl_64` |
| `switch_fpu` (asm/fpu/sched.h) | per-switch: FPU save+TIF_NEED_FPU_LOAD | external |
| `switch_to_extra` (asm/switch_to.h) | per-switch: debug regs + I/O bitmap + xtra | external |
| `update_task_stack` (asm/switch_to.h) | per-switch: TSS sp0 reload | external |

## Compatibility contract

REQ-1: `__switch_to_asm` (assembly, entry_64.S) contract — invoked via the `switch_to` macro:
- Inputs (System V AMD64 ABI): `%rdi = prev_p`, `%rsi = next_p`.
- Pushes callee-saved registers (`%rbp %rbx %r12 %r13 %r14 %r15`) onto prev's kernel stack.
- Saves stack-canary if `CONFIG_STACKPROTECTOR` is set, loads next's canary into `%gs:fixed_percpu_data + stack_canary_offset` before the stack swap.
- `mov %rsp, TASK_threadsp(%rdi)` — store prev's RSP into `prev->thread.sp`.
- `mov TASK_threadsp(%rsi), %rsp` — load next's RSP.
- Pops callee-saved registers from next's kernel stack.
- IBT/CET endbr-aware on jump target.
- `jmp __switch_to` (tail-call C body); the eventual `ret` returns to whatever was on next's stack — `ret_from_fork_asm` for a newly forked task, or the caller of `schedule()` otherwise.

REQ-2: `__switch_to(prev_p, next_p) -> prev_p`:
- `prev = &prev_p->thread`; `next = &next_p->thread`.
- `cpu = smp_processor_id()`.
- `WARN_ON_ONCE(IS_ENABLED(CONFIG_DEBUG_ENTRY) ∧ this_cpu_read(hardirq_stack_inuse))`.
- `switch_fpu(prev_p, cpu)` — sets `TIF_NEED_FPU_LOAD` on next so that the next userspace-return reloads xfpu; also re-enables CR0.TS / XCR0 management.
- `save_fsgs(prev_p)` — must precede `load_TLS` because `load_TLS` may clobber FS/GS (e.g. xen_load_tls).
- `load_TLS(next, cpu)` — write GDT TLS entries `GDT_ENTRY_TLS_MIN..MAX` from `next->thread.tls_array`.
- `arch_end_context_switch(next_p)` — flush paravirt lazy mode.
- DS reload: `savesegment(es, prev->es)`; if `unlikely(next->es | prev->es)`: `loadsegment(es, next->es)`.
- ES reload: `savesegment(ds, prev->ds)`; if `unlikely(next->ds | prev->ds)`: `loadsegment(ds, next->ds)`.
- `x86_fsgsbase_load(prev, next)` (see REQ-7).
- `x86_pkru_load(prev, next)` (see REQ-8).
- PDA: `raw_cpu_write(current_task, next_p)`; `raw_cpu_write(cpu_current_top_of_stack, task_top_of_stack(next_p))`.
- `update_task_stack(next_p)` — TSS sp0 := top-of-stack of next.
- `switch_to_extra(prev_p, next_p)` — debug-registers swap (DR0..DR7) when either side uses HW breakpoints; I/O bitmap update (via `__switch_to_xtra` from process.c) when either side has `TIF_IO_BITMAP`; speculation MSRs etc.
- AMD SYSRET-SS workaround (`X86_BUG_SYSRET_SS_ATTRS`): `savesegment(ss, ss_sel)`; if `ss_sel != __KERNEL_DS`: `loadsegment(ss, __KERNEL_DS)`.
- `resctrl_arch_sched_in(next_p)` — RDT PQR_ASSOC MSR.
- If `X86_FEATURE_AMD_WORKLOAD_CLASS`: `wrmsrl(MSR_AMD_WORKLOAD_HRST, 0x1)` — reset HW history.
- return `prev_p` (RAX for caller of `__switch_to_asm`).

REQ-3: `__rdgsbase_inactive() -> u64` (`noinstr`):
- `lockdep_assert_irqs_disabled()`.
- If `!X86_FEATURE_FRED ∧ !X86_FEATURE_XENPV`:
  - `native_swapgs()`; `gsbase = rdgsbase()`; `native_swapgs()`.
- Else (FRED or Xen-PV):
  - `instrumentation_begin/_end` around `rdmsrq(MSR_KERNEL_GS_BASE, gsbase)`.
- return `gsbase`.

REQ-4: `__wrgsbase_inactive(gsbase)` (`noinstr`):
- `lockdep_assert_irqs_disabled()`.
- If `!X86_FEATURE_FRED ∧ !X86_FEATURE_XENPV`:
  - `native_swapgs()`; `wrgsbase(gsbase)`; `native_swapgs()`.
- Else: `wrmsrq(MSR_KERNEL_GS_BASE, gsbase)` (instrumentation-bracketed).

REQ-5: `save_base_legacy(prev_p, selector, which)` (`__always_inline`):
- If `selector == 0`: leave `prev_p->thread.{fsbase,gsbase}` unchanged (matches historical Linux behavior; relies on AMD `X86_BUG_NULL_SEG` users having already set base via wrmsr).
- Else: clear corresponding `fsbase` or `gsbase` to 0 (selectors 1/2/3 imply base 0 on non-NULL_SEG CPUs; selectors >3 refer to real segments needing no base save).

REQ-6: `save_fsgs(task)` (`__always_inline`):
- `savesegment(fs, task->thread.fsindex)`; `savesegment(gs, task->thread.gsindex)`.
- If `X86_FEATURE_FSGSBASE`: `task->thread.fsbase = rdfsbase()`; `task->thread.gsbase = __rdgsbase_inactive()`.
- Else: `save_base_legacy(task, fsindex, FS)`; `save_base_legacy(task, gsindex, GS)`.

REQ-7: `x86_fsgsbase_load(prev, next)` (`__always_inline`):
- FSGSBASE path:
  - If `prev->fsindex || next->fsindex`: `loadseg(FS, next->fsindex)`.
  - If `prev->gsindex || next->gsindex`: `loadseg(GS, next->gsindex)`.
  - `wrfsbase(next->fsbase)`.
  - `__wrgsbase_inactive(next->gsbase)`.
- Legacy path: `load_seg_legacy(prev->fsindex, prev->fsbase, next->fsindex, next->fsbase, FS)` and same for GS.

REQ-8: `x86_pkru_load(prev, next)` (`__always_inline`):
- Return if `!X86_FEATURE_OSPKE`.
- `prev->pkru = rdpkru()`.
- If `prev->pkru != next->pkru`: `wrpkru(next->pkru)`.

REQ-9: `current_save_fsgs()`:
- `local_irq_save(flags)`; `save_fsgs(current)`; `local_irq_restore(flags)`.
- Exported for KVM (`EXPORT_SYMBOL_FOR_KVM`).

REQ-10: `x86_fsgsbase_read_task(task, selector) -> u64`:
- `idx = selector >> 3`.
- If `(selector & SEGMENT_TI_MASK) == 0` (GDT):
  - If `idx >= GDT_ENTRIES`: return 0.
  - If `idx < GDT_ENTRY_TLS_MIN || idx > GDT_ENTRY_TLS_MAX`: return 0.
  - `idx -= GDT_ENTRY_TLS_MIN`; `base = get_desc_base(&task->thread.tls_array[idx])`.
- Else (LDT, requires `CONFIG_MODIFY_LDT_SYSCALL`):
  - `mutex_lock(&task->mm->context.lock)`.
  - `ldt = task->mm->context.ldt`.
  - If `!ldt || idx >= ldt->nr_entries`: `base = 0`; else `base = get_desc_base(ldt->entries + idx)`.
  - `mutex_unlock`.
- return `base`.

REQ-11: `x86_fsbase_read_task(task)`:
- If `task == current`: `x86_fsbase_read_cpu()`.
- Else if `X86_FEATURE_FSGSBASE || task->thread.fsindex == 0`: return `task->thread.fsbase`.
- Else: `x86_fsgsbase_read_task(task, task->thread.fsindex)`.

REQ-12: `x86_gsbase_read_task(task)`:
- If `task == current`: `x86_gsbase_read_cpu_inactive()`.
- Else if `X86_FEATURE_FSGSBASE || task->thread.gsindex == 0`: return `task->thread.gsbase`.
- Else: `x86_fsgsbase_read_task(task, task->thread.gsindex)`.

REQ-13: `x86_fsbase_write_task(task, fsbase)`:
- `WARN_ON_ONCE(task == current)` — current must use ARCH_SET_FS / `x86_fsbase_write_cpu`.
- `task->thread.fsbase = fsbase`.

REQ-14: `x86_gsbase_write_task(task, gsbase)`:
- `WARN_ON_ONCE(task == current)`.
- `task->thread.gsbase = gsbase`.

REQ-15: `start_thread_common(regs, ip, sp, cs, ss, ds)`:
- `WARN_ON_ONCE(regs != current_pt_regs())`.
- If `X86_BUG_NULL_SEG`: `loadsegment(fs, __USER_DS)`; `load_gs_index(__USER_DS)` (so the subsequent zero-selector load also clears base).
- `reset_thread_features()` (resets per-thread per-prctl features).
- `loadsegment(fs, 0)`; `loadsegment(es, ds)`; `loadsegment(ds, ds)`; `load_gs_index(0)`.
- `regs->ip = ip`; `regs->sp = sp`; `regs->csx = cs`; `regs->ssx = ss`.
- If `X86_FEATURE_FRED`: `regs->fred_ss.swevent = true`; `regs->fred_ss.nmi = true`.
- `regs->flags = X86_EFLAGS_IF | X86_EFLAGS_FIXED`.

REQ-16: `start_thread(regs, ip, sp)`:
- `start_thread_common(regs, ip, sp, __USER_CS, __USER_DS, 0)`.

REQ-17: `compat_start_thread(regs, ip, sp, x32)`:
- `start_thread_common(regs, ip, sp, x32 ? __USER_CS : __USER32_CS, __USER_DS, __USER_DS)`.

REQ-18: `set_personality_64bit()`:
- `clear_thread_flag(TIF_ADDR32)`.
- `task_pt_regs(current)->orig_ax = __NR_execve`.
- `current_thread_info()->status &= ~TS_COMPAT`.
- If `current->mm`: `__set_bit(MM_CONTEXT_HAS_VSYSCALL, &current->mm->context.flags)`.
- `current->personality &= ~READ_IMPLIES_EXEC`.

REQ-19: `set_personality_ia32(x32)`:
- `set_thread_flag(TIF_ADDR32)`.
- If `x32`: `__set_personality_x32` (clear context flags; set ORIG_RAX = `__NR_x32_execve | __X32_SYSCALL_BIT`; clear TS_COMPAT).
- Else: `__set_personality_ia32` (set `MM_CONTEXT_UPROBE_IA32`; OR `force_personality32`; ORIG_RAX = `__NR_ia32_execve`; set TS_COMPAT).

REQ-20: `do_arch_prctl_64(task, option, arg2)`:
- `ARCH_SET_GS`:
  - If `arg2 >= TASK_SIZE_MAX`: return `-EPERM`.
  - `preempt_disable`.
  - If `task == current`: `loadseg(GS, 0)`; `x86_gsbase_write_cpu_inactive(arg2)`; `task->thread.gsbase = arg2`.
  - Else: `task->thread.gsindex = 0`; `x86_gsbase_write_task(task, arg2)`.
  - `preempt_enable`.
- `ARCH_SET_FS`:
  - If `arg2 >= TASK_SIZE_MAX`: return `-EPERM`.
  - `preempt_disable`.
  - If `task == current`: `loadseg(FS, 0)`; `x86_fsbase_write_cpu(arg2)`; `task->thread.fsbase = arg2`.
  - Else: `task->thread.fsindex = 0`; `x86_fsbase_write_task(task, arg2)`.
  - `preempt_enable`.
- `ARCH_GET_FS`: `put_user(x86_fsbase_read_task(task), (unsigned long __user *)arg2)`.
- `ARCH_GET_GS`: `put_user(x86_gsbase_read_task(task), (unsigned long __user *)arg2)`.
- `ARCH_MAP_VDSO_{X32,32,64}` (`CONFIG_CHECKPOINT_RESTORE`): `prctl_map_vdso(image, arg2)`.
- `ARCH_GET_UNTAG_MASK`: `put_user(task->mm->context.untag_mask, ...)`.
- `ARCH_ENABLE_TAGGED_ADDR`: `prctl_enable_tagged_addr(task->mm, arg2)`.
- `ARCH_FORCE_TAGGED_SVA`: require `current == task`; `set_bit(MM_CONTEXT_FORCE_TAGGED_SVA, ...)`.
- `ARCH_GET_MAX_TAG_BITS`: 0 if no LAM, else `LAM_U57_BITS = 6`.
- `ARCH_SHSTK_{ENABLE,DISABLE,LOCK,UNLOCK,STATUS}`: `shstk_prctl(task, option, arg2)`.
- Default: `-EINVAL`.

REQ-21: `prctl_enable_tagged_addr(mm, nr_bits)`:
- Require `X86_FEATURE_LAM` — else `-ENODEV`.
- Require `current->mm == mm` (rejects PTRACE_ARCH_PRCTL on remote mm) — else `-EINVAL`.
- If `mm_valid_pasid(mm) ∧ !MM_CONTEXT_FORCE_TAGGED_SVA`: `-EINVAL`.
- `mmap_write_lock_killable(mm)` — else `-EINTR`.
- If `MM_CONTEXT_LOCK_LAM` already set (multithreaded): unlock; `-EBUSY`.
- If `!nr_bits ∨ nr_bits > LAM_U57_BITS`: unlock; `-EINVAL`.
- `mm_enable_lam(mm)`:
  - `mm->context.lam_cr3_mask = X86_CR3_LAM_U57`.
  - `mm->context.untag_mask = ~GENMASK(62, 57)`.
  - `on_each_cpu_mask(mm_cpumask(mm), enable_lam_func, mm, true)` — IPI loaded mms.
  - `set_bit(MM_CONTEXT_LOCK_LAM, &mm->context.flags)`.
- Unlock; return 0.

REQ-22: `__show_regs(regs, mode, log_lvl)`:
- `show_iret_regs(regs, log_lvl)`.
- Per-mode: SHORT exits early; USER prints only FS_BASE + KERNEL_GS_BASE; ALL prints DS/ES/FS/GS selectors + FS/GS/shadowGS bases + CS/CR0..CR4 + DR0..DR7 (only when non-default per `DR6_RESERVED`/`DR7_FIXED_1`).
- If `CR4.PKE`: print `read_pkru()`.

REQ-23: `release_thread(dead_task)`:
- `WARN_ON(dead_task->mm)` — final invariant that mm has been torn down.

REQ-24: Kernel stack-canary integration (`__switch_to_asm`):
- When `CONFIG_STACKPROTECTOR` is enabled, `__switch_to_asm` loads `next_p->stack_canary` into the per-CPU `fixed_percpu_data.stack_canary` (accessed as `%gs:STACK_CANARY_OFFSET`) before swapping `rsp`, so functions executing in next's stack frame see the correct canary.

REQ-25: ptrace / syscall-trace integration:
- FS/GS bases visible to ptrace via `x86_fsbase_read_task` / `x86_gsbase_read_task` (REQ-11/12); ptrace must not call `_write_task` on `current` (REQ-13/14).
- `ARCH_GET/SET_FS/GS` permitted on remote `task` via `do_arch_prctl_64` from the `ptrace` machinery; updates `thread.{fs,gs}base` directly without touching live registers.
- syscall-trace stop / `syscall_exit_to_user_mode` is centralized in entry-common; this file does not handle it directly, but `ret_from_fork` (in process.c) chains into it on first userspace transition.

## Acceptance Criteria

- [ ] AC-1: `__switch_to` order: `switch_fpu` precedes `save_fsgs`; `save_fsgs` precedes `load_TLS`; `load_TLS` precedes DS/ES reload; FS/GS-base load precedes PKRU load; PDA write precedes `update_task_stack`; `switch_to_extra` runs last (before SYSRET-SS fixup / resctrl).
- [ ] AC-2: `__switch_to` returns `prev_p` (consumed by `__switch_to_asm` as RAX into the next task's stack context).
- [ ] AC-3: `save_fsgs` on FSGSBASE CPUs reads `fsbase` via `rdfsbase` and `gsbase` via `__rdgsbase_inactive`; on non-FSGSBASE CPUs uses `save_base_legacy`.
- [ ] AC-4: `__rdgsbase_inactive` on non-FRED, non-XENPV path issues exactly `SWAPGS; RDGSBASE; SWAPGS` and is marked `noinstr`.
- [ ] AC-5: `__wrgsbase_inactive` on FRED path uses `wrmsrq(MSR_KERNEL_GS_BASE, ...)` (no SWAPGS).
- [ ] AC-6: `x86_fsgsbase_load` FSGSBASE path: selector loaded iff either side was non-zero; base unconditionally written.
- [ ] AC-7: `x86_pkru_load` writes WRPKRU iff `prev->pkru != next->pkru` and OSPKE is enabled.
- [ ] AC-8: `start_thread` (64-bit): regs->cs = __USER_CS; regs->ss = __USER_DS; flags = IF|FIXED; FS/GS selectors zero; ES/DS = 0.
- [ ] AC-9: `compat_start_thread(x32=true)`: cs = __USER_CS, ss = __USER_DS, ds = __USER_DS; `x32=false`: cs = __USER32_CS.
- [ ] AC-10: `do_arch_prctl_64(ARCH_SET_FS, arg2 >= TASK_SIZE_MAX)` returns `-EPERM`.
- [ ] AC-11: `do_arch_prctl_64(ARCH_SET_GS, ..., task == current)` writes both the live KERNEL_GS_BASE (via `x86_gsbase_write_cpu_inactive`) and `thread.gsbase`, with preempt disabled across both.
- [ ] AC-12: `do_arch_prctl_64(ARCH_GET_FS, ...)` returns `put_user` result; reads via `x86_fsbase_read_task` (dispatches to cpu / thread-cached / GDT-LDT path).
- [ ] AC-13: `x86_fsbase_write_task(current, _)` triggers `WARN_ON_ONCE`.
- [ ] AC-14: `prctl_enable_tagged_addr` rejects with `-EBUSY` if `MM_CONTEXT_LOCK_LAM` already set (i.e. mm is multithreaded).
- [ ] AC-15: `set_personality_64bit`: TIF_ADDR32 cleared, TS_COMPAT cleared, ORIG_RAX = __NR_execve.

## Architecture

```
struct ThreadStruct (64-bit fields touched here) {
  sp:          u64,
  fsbase:      u64,
  gsbase:      u64,   // when scheduled out, this is the user GS base (MSR_KERNEL_GS_BASE)
  fsindex:     u16,
  gsindex:     u16,
  es:          u16, ds: u16,
  tls_array:   [Desc; GDT_ENTRY_TLS_ENTRIES],
  pkru:        u32,
  ptrace_bps:  [*PerfEvent; HBP_NUM],  // touched by switch_to_extra
  debugreg6:   u64, debugreg7: u64,
  /* fpu trailing dynamic state */
}

enum WhichSelector { FS, GS }
```

`arch::process64::switch_to(prev_p, next_p) -> *TaskStruct`:
1. prev = &prev_p.thread; next = &next_p.thread.
2. cpu = smp_processor_id().
3. switch_fpu(prev_p, cpu).
4. save_fsgs(prev_p).
5. load_TLS(next, cpu).
6. arch_end_context_switch(next_p).
7. savesegment(es, prev.es); if (next.es | prev.es) != 0: loadsegment(es, next.es).
8. savesegment(ds, prev.ds); if (next.ds | prev.ds) != 0: loadsegment(ds, next.ds).
9. fsgsbase_load(prev, next).
10. pkru_load(prev, next).
11. raw_cpu_write(current_task, next_p).
12. raw_cpu_write(cpu_current_top_of_stack, task_top_of_stack(next_p)).
13. update_task_stack(next_p).
14. switch_to_extra(prev_p, next_p).    // debug regs, IO bitmap, __switch_to_xtra (process.c)
15. if X86_BUG_SYSRET_SS_ATTRS: savesegment(ss, ss_sel); if ss_sel != __KERNEL_DS: loadsegment(ss, __KERNEL_DS).
16. resctrl_arch_sched_in(next_p).
17. if X86_FEATURE_AMD_WORKLOAD_CLASS: wrmsrl(MSR_AMD_WORKLOAD_HRST, 0x1).
18. return prev_p.

`arch::process64::fsgsbase_load(prev, next)`:
1. if X86_FEATURE_FSGSBASE:
   - if prev.fsindex != 0 ∨ next.fsindex != 0: loadseg(FS, next.fsindex).
   - if prev.gsindex != 0 ∨ next.gsindex != 0: loadseg(GS, next.gsindex).
   - wrfsbase(next.fsbase).
   - wrgsbase_inactive(next.gsbase).
2. else:
   - load_seg_legacy(prev.fsindex, prev.fsbase, next.fsindex, next.fsbase, FS).
   - load_seg_legacy(prev.gsindex, prev.gsbase, next.gsindex, next.gsbase, GS).

`arch::process64::fsgsbase_read_task(task, selector) -> u64`:
1. idx = selector >> 3.
2. if (selector & SEGMENT_TI_MASK) == 0:                // GDT
   - if idx >= GDT_ENTRIES: return 0.
   - if idx < GDT_ENTRY_TLS_MIN ∨ idx > GDT_ENTRY_TLS_MAX: return 0.
   - return get_desc_base(&task.thread.tls_array[idx - GDT_ENTRY_TLS_MIN]).
3. else:                                                // LDT
   - mutex_lock(&task.mm.context.lock).
   - ldt = task.mm.context.ldt.
   - base = (ldt && idx < ldt.nr_entries) ? get_desc_base(ldt.entries + idx) : 0.
   - mutex_unlock; return base.

`arch::process64::do_arch_prctl_64(task, option, arg2) -> i64`:
1. ARCH_SET_GS / ARCH_SET_FS: validate arg2 < TASK_SIZE_MAX; preempt_disable; current-vs-non-current split (loadseg zero + write_cpu_inactive vs write_task); preempt_enable.
2. ARCH_GET_FS / ARCH_GET_GS: put_user(read_task(...)).
3. ARCH_MAP_VDSO_{32,X32,64}: prctl_map_vdso(image, arg2).
4. ARCH_GET_UNTAG_MASK: put_user(mm.context.untag_mask).
5. ARCH_ENABLE_TAGGED_ADDR: prctl_enable_tagged_addr(task.mm, arg2).
6. ARCH_FORCE_TAGGED_SVA: require current == task; set MM_CONTEXT_FORCE_TAGGED_SVA.
7. ARCH_GET_MAX_TAG_BITS: 0 or LAM_U57_BITS.
8. ARCH_SHSTK_*: shstk_prctl(task, option, arg2).
9. default: -EINVAL.

`arch::process64::start_thread(regs, ip, sp)`:
1. start_thread_common(regs, ip, sp, __USER_CS, __USER_DS, 0):
   - WARN_ON_ONCE(regs != current_pt_regs()).
   - if X86_BUG_NULL_SEG: loadsegment(fs, __USER_DS); load_gs_index(__USER_DS).
   - reset_thread_features().
   - loadsegment(fs, 0); loadsegment(es, ds_arg); loadsegment(ds, ds_arg); load_gs_index(0).
   - regs.ip = ip; regs.sp = sp; regs.csx = cs; regs.ssx = ss.
   - if FRED: regs.fred_ss.swevent = true; regs.fred_ss.nmi = true.
   - regs.flags = X86_EFLAGS_IF | X86_EFLAGS_FIXED.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `switch_to_ordering` | INVARIANT | per-__switch_to: switch_fpu < save_fsgs < load_TLS < fsgsbase_load < pkru_load < per-CPU write < update_task_stack < switch_to_extra. |
| `rdgsbase_inactive_irq_off` | INVARIANT | per-__rdgsbase_inactive: lockdep IRQs-off. |
| `wrgsbase_inactive_irq_off` | INVARIANT | per-__wrgsbase_inactive: lockdep IRQs-off. |
| `swapgs_pairing` | INVARIANT | per-__rdgsbase/__wrgsbase_inactive non-FRED path: exactly two `native_swapgs()` calls; matched. |
| `fsbase_write_task_not_current` | INVARIANT | per-x86_fsbase_write_task: WARN_ON_ONCE(task == current). |
| `gsbase_write_task_not_current` | INVARIANT | per-x86_gsbase_write_task: WARN_ON_ONCE(task == current). |
| `arch_set_fs_below_task_size_max` | INVARIANT | per-ARCH_SET_FS: arg2 < TASK_SIZE_MAX else -EPERM. |
| `arch_set_gs_below_task_size_max` | INVARIANT | per-ARCH_SET_GS: arg2 < TASK_SIZE_MAX else -EPERM. |
| `lam_enable_requires_single_thread` | INVARIANT | per-prctl_enable_tagged_addr: MM_CONTEXT_LOCK_LAM not set on entry (else -EBUSY). |
| `start_thread_eflags_fixed_if` | INVARIANT | per-start_thread_common: regs.flags == (X86_EFLAGS_IF | X86_EFLAGS_FIXED). |
| `fsgsbase_load_base_always_written_fsgsbase` | INVARIANT | per-x86_fsgsbase_load (FSGSBASE): wrfsbase and wrgsbase_inactive always run. |

### Layer 2: TLA+

`arch/x86/process-64.tla`:
- Models the switch as a single atomic action `Switch(prev, next)` decomposed into ordered substeps with per-CPU live state {current_task, top_of_stack, TSS.sp0, FS_BASE, KERNEL_GS_BASE, FS_SEL, GS_SEL, DS, ES, PKRU, DR0..DR7, IO_BITMAP}.
- Properties:
  - `safety_per_cpu_current_after_load_TLS` — per-switch: `current_task` per-CPU write happens-after `load_TLS` to avoid stale TLS during paravirt callbacks.
  - `safety_TSS_sp0_matches_next_top` — per-switch: after `update_task_stack`, TSS.sp0 == task_top_of_stack(next_p).
  - `safety_fsgs_load_after_save_prev` — per-switch: `save_fsgs(prev)` happens-before `fsgsbase_load(prev, next)`.
  - `safety_swapgs_balanced` — per-__rdgsbase_inactive / __wrgsbase_inactive non-FRED: SWAPGS count is even on all paths.
  - `safety_arch_set_fs_user_addr` — per-ARCH_SET_FS: arg2 < TASK_SIZE_MAX always.
  - `safety_arch_set_gs_user_addr` — per-ARCH_SET_GS: arg2 < TASK_SIZE_MAX always.
  - `safety_no_kernel_gs_leak_to_user` — per-userret: KERNEL_GS_BASE never holds a kernel-address while CPL==3 (composed with entry/exit Tier-3).
  - `liveness_switch_completes` — per-Switch(prev,next): all substeps execute; PDA pointer eventually points at next.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `__switch_to` post: current_task per-CPU == next_p; cpu_current_top_of_stack == task_top_of_stack(next_p) | `__switch_to` |
| `__switch_to` post: TSS.sp0 == task_top_of_stack(next_p) | `__switch_to → update_task_stack` |
| `save_fsgs` post: thread.fsindex/gsindex reflect actual selectors; thread.fsbase/gsbase reflect either MSR/RDFSBASE or legacy heuristic | `save_fsgs` |
| `x86_fsgsbase_load` post: FS_BASE MSR == next.fsbase; KERNEL_GS_BASE MSR == next.gsbase | `x86_fsgsbase_load` |
| `x86_pkru_load` post: hw PKRU == next.pkru; prev.pkru == old hw PKRU | `x86_pkru_load` |
| `do_arch_prctl_64(ARCH_SET_FS, current, base)` post: thread.fsbase == base ∧ FS_BASE MSR == base ∧ fsindex == 0 | `do_arch_prctl_64` |
| `do_arch_prctl_64(ARCH_SET_GS, current, base)` post: thread.gsbase == base ∧ KERNEL_GS_BASE MSR == base ∧ gsindex == 0 | `do_arch_prctl_64` |
| `do_arch_prctl_64(ARCH_GET_FS, task, uaddr)` post: *uaddr == x86_fsbase_read_task(task) | `do_arch_prctl_64` |
| `prctl_enable_tagged_addr` post (success): mm.context.lam_cr3_mask == X86_CR3_LAM_U57; mm.context.untag_mask == ~GENMASK(62,57); MM_CONTEXT_LOCK_LAM set | `prctl_enable_tagged_addr` |
| `start_thread` post: regs.cs == __USER_CS; regs.ss == __USER_DS; FS/GS selectors and bases cleared | `start_thread` |
| `set_personality_64bit` post: TIF_ADDR32 clear; TS_COMPAT clear; orig_ax == __NR_execve | `set_personality_64bit` |

### Layer 4: Verus/Creusot functional

`schedule() → __switch_to_asm → __switch_to(prev, next) → return → ret in next's frame`:
- Semantic equivalence to the C path: prev's RSP captured into prev.thread.sp; next.thread.sp loaded into RSP; callee-saved regs of next restored; stack-canary of next loaded into per-CPU; control returns into whatever code next had pushed (either ret_from_fork_asm or post-schedule()).

`ARCH_SET_FS / ARCH_SET_GS path (current)`:
- Equivalence to `glibc set_thread_area` / `set_thread_pointer` expectations: subsequent FS-relative or GS-relative accesses observe the new base immediately (single-CPU); on context switch out and back in, base is reloaded via `x86_fsgsbase_load`.

`PTRACE_ARCH_PRCTL on remote task`:
- `do_arch_prctl_64(remote_task, ARCH_SET_FS, base)` mutates only `remote_task.thread.fsbase` and zeros `fsindex`; live MSR is *not* touched (consistent with REQ-13/14 WARN). Next time `remote_task` schedules in, `x86_fsgsbase_load` writes the new base.

`LAM enable path`:
- `prctl(ARCH_ENABLE_TAGGED_ADDR, 6)` from a single-threaded process → CR3 LAM bit set on this CPU and IPI-broadcast to all CPUs running this mm; subsequent loads with tagged pointers strip bits 62:57.

## Hardening

(Inherits arch baseline from `arch/x86/00-overview.md` § Hardening.)

x86_64 switch-and-prctl reinforcement:

- **`__rdgsbase_inactive` / `__wrgsbase_inactive` marked `noinstr`** — defense against per-trace recursion accessing per-CPU data with the wrong GS.
- **`lockdep_assert_irqs_disabled` in both inactive-GS-base helpers** — defense against per-NMI-mid-SWAPGS catastrophe.
- **FRED / Xen-PV special-cased: no SWAPGS** — defense against per-FRED-incompatible swapgs trap.
- **`save_fsgs(prev)` runs before `load_TLS(next)`** — defense against per-paravirt-load_TLS clobbering FS/GS we forgot to save.
- **`fsgsbase_load` writes base unconditionally on FSGSBASE CPUs** — defense against per-stale-base-from-zero-selector-leak.
- **`x86_pkru_load` gated on diff** — defense against per-context-switch WRPKRU latency.
- **`do_arch_prctl_64` validates `arg2 < TASK_SIZE_MAX` for SET_FS/SET_GS** — defense against per-user-base-pointing-at-kernel-canonical-hole exploit.
- **`x86_fsbase_write_task` / `x86_gsbase_write_task` refuse `task == current`** — defense against per-ptrace-self confusion (must use SET_FS path).
- **`prctl_enable_tagged_addr` requires single-threaded mm** — defense against per-mid-LAM-toggle racing thread.
- **`prctl_enable_tagged_addr` IPI all CPUs running mm before set_bit** — defense against per-stale-CR3 partial-LAM state.
- **`update_task_stack` mandatory before user return** — defense against per-stale-TSS-sp0 exploit (kernel return to wrong stack).
- **AMD `X86_BUG_SYSRET_SS_ATTRS` SS=__KERNEL_DS fixup** — defense against per-SYSRET-with-NULL-SS-unusable-cached-descriptor crash.
- **`switch_fpu` first in `__switch_to`** — defense against per-fault-during-switch leaving FPU state cross-contaminated.
- **`WARN_ON_ONCE(hardirq_stack_inuse)` on switch entry** — defense against per-buggy IRQ-stack-unwind invoking schedule().

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded `put_user` / `get_user` on `ARCH_GET_FS`, `ARCH_GET_GS`, `ARCH_GET_UNTAG_MASK`, `ARCH_GET_MAX_TAG_BITS`; rejects buffers crossing kernel/user boundary.
- **PAX_KERNEXEC** — W^X on the `__switch_to_asm` / `__switch_to` text and FRED/SWAPGS trampolines; any write attempt via stray pointer faults.
- **PAX_RANDKSTACK** — per-syscall kernel-stack offset randomization applied before each `do_arch_prctl_64` entry; randomized top-of-stack stored back into `cpu_current_top_of_stack` and TSS.sp0.
- **PAX_REFCOUNT** — saturating refcount on `task_struct` (`usage`, `stack_refcount`) to prevent UAF on `__switch_to(prev)` after a freed `task_struct`.
- **PAX_MEMORY_STACKLEAK** — kernel stack erased on `__switch_to_asm` return path so leftover `thread.fsbase`/`gsbase` shadow stack residues never leak into the next task.
- **PAX_MEMORY_SANITIZE** — `thread_struct` slab poisoned on `release_thread`; FS/GS base, PKRU, debug registers, and TLS array zeroed before slab reuse.
- **PAX_UDEREF** — SMAP/SMEP strictly enforced when reading user pointers in `ARCH_GET_*` (uaccess windows bracketed by `stac`/`clac`).
- **PAX_RAP / kCFI** — indirect-call signatures on paravirt callbacks (`arch_end_context_switch`, `xen_load_tls`) and on `shstk_prctl` dispatch from `do_arch_prctl_64`.
- **GRKERNSEC_HIDESYM** — kernel pointers and FS_BASE / KERNEL_GS_BASE redacted in `__show_regs(USER)`; `/proc/<pid>/{stat,maps,syscall}` exposes no thread-side stack/base info to non-CAP_SYSLOG readers.
- **GRKERNSEC_DMESG** — oops `__show_regs(ALL)` printk gated behind `dmesg_restrict`.
- **GRKERNSEC_PROC** — `/proc/<pid>/arch_status` and per-thread FS/GS-base readout require matching uid or CAP_SYS_PTRACE.
- **GRKERNSEC_PTRACE** — `do_arch_prctl_64(remote_task, ARCH_SET_FS/GS)` checked under ptrace-scope policy before mutating `thread.{fs,gs}base`.

Per-doc rationale: `__switch_to` is the single point where every task's privileged state (FS/GS bases, KERNEL_GS_BASE, PKRU, TSS.sp0, debug registers, speculation MSRs) is mutated; a single missed sanitization or pointer-leak here turns into a kernel-wide info leak or privilege confusion, so the PaX surface is concentrated on USERCOPY-bounded prctl get-paths, RAP-protected paravirt vtables, and STACKLEAK on the post-switch return path.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `arch/x86/entry/entry_64.S` — `__switch_to_asm`, `ret_from_fork_asm`, SYSCALL/SYSRET, IDT entries (covered in `arch/x86/entry.md`).
- `arch/x86/kernel/process.c` — generic task lifecycle, idle loop, `__switch_to_xtra`, 32-bit arch_prctl front-end (covered in `arch/x86/process.md`).
- `arch/x86/kernel/fpu/*` — FPU save/restore internals, dynamic XSAVE (covered in `arch/x86/fpu.md`).
- `arch/x86/kernel/shstk.c` — `shstk_prctl`, shadow stack allocator (covered in `arch/x86/shstk.md`).
- `arch/x86/kernel/io_bitmap.c` — IO bitmap allocation / TSS copy (covered in `arch/x86/io-bitmap.md`).
- `arch/x86/kernel/ldt.c` — LDT lifecycle (covered in `arch/x86/ldt.md`).
- `arch/x86/kernel/process_32.c` — 32-bit-only switch (out of scope: Rookery is 64-bit-first).
- `kernel/sched/*` — generic scheduler / `schedule()` (covered in `sched-core.md`).
- `arch/x86/kernel/resctrl/*` — RDT MSR programming (covered separately if expanded).
- Implementation code.
