---
title: "Tier-3: arch/x86/kernel/traps.c — x86 hardware traps and exception handlers"
tags: ["tier-3", "arch", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`arch/x86/kernel/traps.c` is the **central dispatcher** for x86 hardware exceptions: each architectural fault (`#DE`, `#DB`, `#BP`, `#OF`, `#BR`, `#UD`, `#NM`, `#DF`, `#TS`, `#NP`, `#SS`, `#GP`, `#MF`, `#AC`, `#XF`, `#VE`) and its FRED / IDT entry-point shim is declared with the `DEFINE_IDTENTRY*` macro family and lands in this file (`#PF` page-faults live in `arch/x86/mm/fault.c`; `#NMI` in `arch/x86/kernel/nmi.c`; `#MC` in `arch/x86/kernel/cpu/mce/core.c`; `#CP` in `arch/x86/kernel/cet.c`; `#VC` in `arch/x86/kernel/sev.c`). Owns: per-trap dispatcher (`do_error_trap` → `do_trap` → `do_trap_no_signal` → `force_sig` / `force_sig_fault` / `die`), per-#UD bug-decode (`decode_bug`, `handle_bug` — emulates `UD2`/`UD1` as `__WARN_trap` / UBSan trap / static_call / CFI / FineIBT failure), per-#DF double-fault handler on IST (ESPFIX64 #GP synthesis, VMAP_STACK guard-page diagnosis, `die("double fault")` → `panic`), per-#DB debug handler split between IST kernel-mode (`exc_debug_kernel` — saves DR7, suppresses recursion) and user-mode (`exc_debug_user`), per-#AC alignment-check with split-lock pass-through (`handle_user_split_lock`), per-#NM device-not-available with XFD (extended-feature-disable) on-demand activation + math-emulation fallback, per-#BP int3 (text-poke + KGDB + kprobes hand-off), per-#GP general-protection with `ENQCMD` PASID fix-up + IOPL emulation + LASS / non-canonical / NULL hinting, per-#VE virtualization-exception (TDX guest VE info fetch + `tdx_handle_virt_exception`), and per-`trap_init()` boot-time IDT/IST install via `idt_setup_traps()` after `cpu_init_exception_handling(true)`. Critical for: every user-mode fault → SIGFPE/SIGSEGV/SIGILL/SIGBUS/SIGTRAP delivery, every kernel-mode oops/die path, double-fault containment, single-step debugging, WARN()/BUG() printing, KPROBES/KGDB hook surface.

This Tier-3 covers `arch/x86/kernel/traps.c` (~1698 lines).

### Acceptance Criteria

- [ ] AC-1: User-mode #DE delivers `SIGFPE` with `si_code == FPE_INTDIV` and `si_addr == regs->ip` (uprobe-aware via `uprobe_get_trap_addr`).
- [ ] AC-2: Kernel-mode #DE without an extable entry triggers `die("divide error")` and oops.
- [ ] AC-3: #UD on user-mode `ud2` delivers `SIGILL/ILL_ILLOPN`; kernel-mode `ud2` from `WARN()` macro returns from `handle_bug` after incrementing `regs->ip` by `ud_len`.
- [ ] AC-4: `decode_bug` distinguishes BUG_UD1_WARN (`ud1 (%edx),%reg`), BUG_UD1_UBSAN (`ud1 (%eax),...`), BUG_UD2 (`0f 0b`), BUG_UDB (`d6`), BUG_LOCK (`f0 75 f9`).
- [ ] AC-5: #DF on a guard-page stack-overflow (VMAP_STACK) prints `"BUG: <stack> stack guard page was hit"` and panics.
- [ ] AC-6: ESPFIX64: a #DF triggered by an IRET fault while on the ESPFIX stack is converted to a synthesized `#GP(0)` and returned to `asm_exc_general_protection`.
- [ ] AC-7: #GP from user-mode `ENQCMD` with valid mm PASID and `current->pasid_activated == 0` returns successfully after writing `MSR_IA32_PASID`.
- [ ] AC-8: #GP from kernel-mode at a fixable RIP returns via `fixup_exception`; otherwise oopses with hint (`"non-canonical address ..."`, `"NULL pointer dereference"`, `"LASS violation for address ..."`, or `"maybe for address ..."`).
- [ ] AC-9: #BP from `text_poke_bp` rewrite is consumed by `smp_text_poke_int3_handler` without entering irqentry; ordinary `int3` from user-mode delivers `SIGTRAP`.
- [ ] AC-10: #DB from kernel uses IST 4 stack; `local_db_save()` masks recursive HW-bp; `DR6_RESERVED` written back after read.
- [ ] AC-11: #DB from user-mode delivers `SIGTRAP` with `si_code = TRAP_TRACE/TRAP_HWBKPT/TRAP_BRKPT` per DR6 bits; icebp (dr6 == 0) delivers SIGTRAP.
- [ ] AC-12: #AC in kernel mode dies with `"Split lock detected"`; user-mode hits `handle_user_split_lock` policy (warn/kill/ratelimit) before SIGBUS.
- [ ] AC-13: #NM with XFD_ERR set + user-mode + permitted feature: `xfd_enable_feature` succeeds and returns; `-EPERM` delivers `SIGILL/ILL_ILLOPC`, `-EFAULT` delivers `SIGSEGV`.
- [ ] AC-14: #NM with CR0.TS set warns "CR0.TS was set" and clears it (no SIGFPE).
- [ ] AC-15: #MF/#XF synchronize FPU state via `fpu_sync_fpstate` before `fpu__exception_code` and deliver `SIGFPE` with the decoded si_code.
- [ ] AC-16: #VE (TDX guest) calls `tdx_get_ve_info` first; on failure to handle, escalates via `ve_raise_fault` → `gp_user_force_sig_segv` (user) or `die_addr` (kernel).
- [ ] AC-17: `trap_init()` is called once at boot, sets up cpu_entry_areas before IDT install, and skips `idt_setup_traps()` when FRED is enabled.

### Architecture

```
const X86_TRAP_DE: u8 = 0;
const X86_TRAP_DB: u8 = 1;
const X86_TRAP_NMI: u8 = 2;
const X86_TRAP_BP: u8 = 3;
const X86_TRAP_OF: u8 = 4;
const X86_TRAP_BR: u8 = 5;
const X86_TRAP_UD: u8 = 6;
const X86_TRAP_NM: u8 = 7;
const X86_TRAP_DF: u8 = 8;
const X86_TRAP_TS: u8 = 10;
const X86_TRAP_NP: u8 = 11;
const X86_TRAP_SS: u8 = 12;
const X86_TRAP_GP: u8 = 13;
const X86_TRAP_PF: u8 = 14;
const X86_TRAP_MF: u8 = 16;
const X86_TRAP_AC: u8 = 17;
const X86_TRAP_MC: u8 = 18;
const X86_TRAP_XF: u8 = 19;
const X86_TRAP_VE: u8 = 20;
const X86_TRAP_CP: u8 = 21;
const X86_TRAP_VC: u8 = 29;

enum KernelGpHint {
  NoHint,
  NonCanonical,
  Canonical,
  LassViolation,
  NullPointer,
}

enum BugType {
  None,
  Ud2,
  Udb,
  Lock,
  Ud1Warn,
  Ud1Ubsan,
}
```

`Trap::do_trap_no_signal(tsk, trapnr, str, regs, error_code) -> i32`:
1. /* vm86 forward for traps 0..5 */
2. If v8086_mode(regs) ∧ trapnr < X86_TRAP_UD ∧ handle_vm86_trap consumed: return 0.
3. If !user_mode(regs):
   - if fixup_exception(regs, trapnr, error_code, 0): return 0.
   - tsk.thread.{error_code,trap_nr} = ...; die(str, regs, error_code).
4. Else if fixup_vdso_exception(regs, trapnr, error_code, 0): return 0.
5. tsk.thread.{error_code,trap_nr} = ...; return -1.

`Trap::do_trap(trapnr, signr, str, regs, error_code, sicode, addr)`:
1. If Trap::do_trap_no_signal(current, ...) == 0: return.
2. Trap::show_signal(current, signr, "trap ", str, regs, error_code).
3. if sicode == 0: force_sig(signr); else force_sig_fault(signr, sicode, addr).

`Trap::handle_bug(regs) -> bool`:
1. ud_type = Trap::decode_bug(regs.ip, &mut imm, &mut len).
2. If ud_type == BugType::None: return false.
3. instrumentation_begin; kmsan_unpoison_entry_regs.
4. Restore IF if X86_EFLAGS_IF was set at fault site (raw_local_irq_enable).
5. Switch ud_type:
   - Ud1Warn: handled = report_bug_entry(pt_regs_val(regs, imm), regs) == BUG_TRAP_TYPE_WARN.
   - Ud2: handled = report_bug(regs.ip, regs) == BUG_TRAP_TYPE_WARN; else fallthrough.
   - Udb | Lock: handled = handle_cfi_failure(regs) == BUG_TRAP_TYPE_WARN.
   - Ud1Ubsan: pr_crit(report_ubsan_failure(imm)) if CONFIG_UBSAN_TRAP.
6. If handled ∧ regs.ip == addr: regs.ip += len. Else: regs.ip = addr.
7. raw_local_irq_disable if was-set; instrumentation_end; return handled.

`Trap::exc_double_fault(regs, error_code) /* IST 1 */`:
1. /* ESPFIX64 #GP synthesis */
2. If on espfix stack ∧ regs.cs == __KERNEL_CS ∧ regs.ip == native_irq_return_iret:
   - Build #GP(0) frame at TSS.sp0; rewrite regs.ip = asm_exc_general_protection; regs.sp = &gpregs.orig_ax; return.
3. irqentry_nmi_enter; notify_die(DIE_TRAP, "double fault", ..., SIGSEGV).
4. tsk.thread.{error_code,trap_nr} = (ec, DF).
5. /* VMAP_STACK guard-page diagnosis */
6. address = read_cr2(); if get_stack_guard_info(address, &info): Trap::handle_stack_overflow(regs, address, &info) /* noreturn */.
7. pr_emerg("PANIC: double fault, error_code: ..."); die("double fault", ...); panic("Machine halted.").

`Trap::exc_general_protection(regs, error_code)`:
1. /* ENQCMD fast-path */
2. If user_mode(regs) ∧ Trap::try_fixup_enqcmd_gp(): return.
3. cond_local_irq_enable.
4. If v8086_mode(regs): handle_vm86_fault(...); return.
5. If user_mode(regs):
   - Trap::fixup_iopl_exception → exit.
   - fixup_vdso_exception(X86_TRAP_GP) → exit.
   - fixup_umip_exception → exit.
   - emulate_vsyscall_gp → exit.
   - Trap::gp_user_force_sig_segv(regs, X86_TRAP_GP, error_code, "general protection fault") → exit.
6. Trap::gp_try_fixup_and_notify(regs, X86_TRAP_GP, error_code, desc, 0) → exit.
7. Build description: segment-related vs hinted address.
8. die_addr(desc, regs, error_code, hint == NonCanonical ? gp_addr : 0).

`Trap::exc_int3(regs) /* DEFINE_IDTENTRY_RAW */`:
1. If smp_text_poke_int3_handler(regs): return /* text-poke consume */.
2. If user_mode(regs):
   - irqentry_enter_from_user_mode; instrumentation_begin.
   - if !do_int3(regs): cond_local_irq_enable; do_trap(X86_TRAP_BP, SIGTRAP, "int3", regs, 0, 0, NULL); cond_local_irq_disable.
   - instrumentation_end; irqentry_exit_to_user_mode.
3. Else:
   - irq_state = irqentry_nmi_enter(regs); instrumentation_begin.
   - if !do_int3(regs): die("int3", regs, 0).
   - instrumentation_end; irqentry_nmi_exit(regs, irq_state).

`Trap::exc_debug_kernel(regs, dr6) /* IST 4 */`:
1. dr7 = local_db_save() /* disable HW breakpoints during handler */.
2. irq_state = irqentry_nmi_enter.
3. WARN_ON_ONCE(user_mode(regs)).
4. If TIF_BLOCKSTEP: re-arm DEBUGCTLMSR_BTF.
5. If !FRED ∧ (dr6 & DR_STEP) ∧ Trap::is_sysenter_singlestep(regs): dr6 &= ~DR_STEP.
6. If !dr6: goto out.
7. If Trap::notify_debug(regs, &dr6): goto out.
8. If dr6 & DR_STEP: WARN+clear regs.flags TF.
9. out: instrumentation_end; irqentry_nmi_exit(regs, irq_state); local_db_restore(dr7).

`Trap::exc_debug_user(regs, dr6)`:
1. WARN_ON_ONCE(!user_mode(regs)).
2. irqentry_enter_from_user_mode; instrumentation_begin.
3. current.thread.virtual_dr6 = dr6 & DR_STEP.
4. clear_thread_flag(TIF_BLOCKSTEP).
5. icebp = (dr6 == 0).
6. If Trap::notify_debug(regs, &dr6): goto out.
7. local_irq_enable.
8. If v8086_mode(regs): handle_vm86_trap(.., 0, X86_TRAP_DB); goto out_irq.
9. If dr6 & DR_BUS_LOCK: handle_bus_lock(regs).
10. dr6 |= current.thread.virtual_dr6.
11. If (dr6 & (DR_STEP | DR_TRAP_BITS)) ∨ icebp: send_sigtrap(regs, 0, get_si_code(dr6)).
12. out_irq: local_irq_disable; out: instrumentation_end; irqentry_exit_to_user_mode.

`Trap::exc_alignment_check(regs, error_code)`:
1. If notify_die(DIE_TRAP, "alignment check", ..., X86_TRAP_AC, SIGBUS) == NOTIFY_STOP: return.
2. If !user_mode(regs): die("Split lock detected\n", regs, error_code).
3. local_irq_enable.
4. if handle_user_split_lock(regs, error_code): goto out.
5. do_trap(X86_TRAP_AC, SIGBUS, "alignment check", regs, error_code, BUS_ADRALN, NULL).
6. out: local_irq_disable.

`Trap::exc_device_not_available(regs)`:
1. cr0 = read_cr0().
2. If Trap::handle_xfd_event(regs): return.
3. /* MATH_EMULATION */
4. If !X86_FEATURE_FPU ∧ (cr0 & X86_CR0_EM): cond_local_irq_enable; math_emulate(&info); cond_local_irq_disable; return.
5. If cr0 & X86_CR0_TS: WARN + write_cr0(cr0 & ~X86_CR0_TS).
6. Else: die("unexpected #NM exception", regs, 0).

`Trap::trap_init()` — `__init`:
1. setup_cpu_entry_areas().
2. sev_es_init_vc_handling().
3. cpu_init_exception_handling(true) /* TSS + IST */.
4. if !X86_FEATURE_FRED: idt_setup_traps().
5. cpu_init().

`Trap::exc_virtualization_exception(regs) /* CONFIG_INTEL_TDX_GUEST */`:
1. tdx_get_ve_info(&ve) /* unblocks NMIs/Interrupts at TDX module */.
2. cond_local_irq_enable.
3. If !tdx_handle_virt_exception(regs, &ve): Trap::ve_raise_fault(regs, 0, ve.gla).
4. cond_local_irq_disable.

### Out of Scope

- arch/x86/mm/fault.c — #PF page-fault handler (covered in `arch/x86/mm/fault.md` Tier-3 when expanded).
- arch/x86/kernel/nmi.c — #NMI handler (covered in `arch/x86/nmi.md` Tier-3 when expanded).
- arch/x86/kernel/cpu/mce/* — #MC machine-check handling (covered in `arch/x86/mce.md` Tier-3 when expanded).
- arch/x86/kernel/cet.c — #CP control-protection (CET) handling (covered in `arch/x86/cet.md` Tier-3 when expanded).
- arch/x86/kernel/sev.c — #VC SEV-ES VMM communication (covered in `arch/x86/sev.md` Tier-3 when expanded).
- arch/x86/kernel/idt.c — IDT table layout and `idt_setup_traps()` body (covered in `arch/x86/00-overview.md`).
- arch/x86/kernel/process*.c — task-switch error_code/trap_nr clearing on fork/exec (covered in `arch/x86/process.md` Tier-3).
- arch/x86/entry/* — `asm_exc_*` thunks and IST stack-switch assembly (covered in `arch/x86/entry.md` Tier-3).
- KGDB / kprobes notifier internals (separate Tier-3).
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `do_trap_no_signal()` | per-trap fix-up vs die vs signal triage | `Trap::do_trap_no_signal` |
| `do_trap()` | per-trap signal-delivery wrapper | `Trap::do_trap` |
| `do_error_trap()` | per-trap notifier + cond_local_irq glue | `Trap::do_error_trap` |
| `show_signal()` | per-signal ratelimited klog | `Trap::show_signal` |
| `error_get_trap_addr()` | per-uprobe-aware si_addr resolve | `Trap::error_get_trap_addr` |
| `decode_bug()` | per-#UD UD1/UD2/UDB/LOCK classify | `Trap::decode_bug` |
| `is_valid_bugaddr()` | per-#UD kernel-addr UD2 check | `Trap::is_valid_bugaddr` |
| `handle_bug()` | per-#UD WARN/BUG/UBSan/CFI dispatch | `Trap::handle_bug` |
| `handle_invalid_op()` | per-#UD user SIGILL fallback | `Trap::handle_invalid_op` |
| `exc_divide_error` (#DE) | per-#DE → SIGFPE/FPE_INTDIV | `Trap::exc_divide_error` |
| `exc_overflow` (#OF) | per-#OF (INTO) → SIGSEGV | `Trap::exc_overflow` |
| `exc_bounds` (#BR) | per-#BR → SIGSEGV | `Trap::exc_bounds` |
| `exc_invalid_op` (#UD) | per-#UD raw entry | `Trap::exc_invalid_op` |
| `exc_coproc_segment_overrun` | per-OLD_MF → SIGFPE | `Trap::exc_coproc_segment_overrun` |
| `exc_invalid_tss` (#TS) | per-#TS → SIGSEGV | `Trap::exc_invalid_tss` |
| `exc_segment_not_present` (#NP) | per-#NP → SIGBUS | `Trap::exc_segment_not_present` |
| `exc_stack_segment` (#SS) | per-#SS → SIGBUS | `Trap::exc_stack_segment` |
| `exc_alignment_check` (#AC) | per-#AC + split-lock | `Trap::exc_alignment_check` |
| `exc_double_fault` (#DF) | per-#DF on IST + ESPFIX64 + stack-guard | `Trap::exc_double_fault` |
| `exc_general_protection` (#GP) | per-#GP + ENQCMD + IOPL + LASS hint | `Trap::exc_general_protection` |
| `exc_int3` (#BP) | per-#BP raw entry (text-poke + KGDB + kprobes) | `Trap::exc_int3` |
| `exc_debug` (#DB) | per-#DB IDT IST entry | `Trap::exc_debug` |
| `exc_coprocessor_error` (#MF) | per-#MF → SIGFPE | `Trap::exc_coprocessor_error` |
| `exc_simd_coprocessor_error` (#XF) | per-#XF → SIGFPE (with 486 INVD fix) | `Trap::exc_simd_coprocessor_error` |
| `exc_spurious_interrupt_bug` | per-PPro erratum 15 | `Trap::exc_spurious_interrupt_bug` |
| `exc_device_not_available` (#NM) | per-#NM + XFD + math-emu | `Trap::exc_device_not_available` |
| `exc_virtualization_exception` (#VE) | per-#VE (TDX guest) | `Trap::exc_virtualization_exception` |
| `iret_error` | 32-bit only IRET fault | `Trap::iret_error` |
| `handle_stack_overflow()` | VMAP_STACK guard-page die() | `Trap::handle_stack_overflow` |
| `exc_debug_kernel()` / `exc_debug_user()` | per-#DB inner | `Trap::exc_debug_kernel` / `_user` |
| `debug_read_reset_dr6()` | per-#DB DR6 read+reset (DR6_RESERVED) | `Trap::debug_read_reset_dr6` |
| `notify_debug()` | per-#DB notifier hand-off | `Trap::notify_debug` |
| `is_sysenter_singlestep()` | per-SYSENTER TF spurious filter | `Trap::is_sysenter_singlestep` |
| `math_error()` | per-#MF/#XF FP/SIMD common | `Trap::math_error` |
| `handle_xfd_event()` | per-#NM XFD fault demand | `Trap::handle_xfd_event` |
| `try_fixup_enqcmd_gp()` | per-#GP PASID MSR populate | `Trap::try_fixup_enqcmd_gp` |
| `fixup_iopl_exception()` | per-#GP CLI/STI NOP emulation | `Trap::fixup_iopl_exception` |
| `get_kernel_gp_address()` | per-#GP RIP decode + hint | `Trap::get_kernel_gp_address` |
| `gp_try_fixup_and_notify()` | per-#GP fix-up + DIE_GPF notify | `Trap::gp_try_fixup_and_notify` |
| `gp_user_force_sig_segv()` | per-#GP user SIGSEGV deliver | `Trap::gp_user_force_sig_segv` |
| `ve_raise_fault()` | per-#VE escalation (TDX) | `Trap::ve_raise_fault` |
| `sync_regs()` | per-IST → kernel-stack copy | `Trap::sync_regs` |
| `vc_switch_off_ist()` | per-#VC IST off-switch | `Trap::vc_switch_off_ist` |
| `fixup_bad_iret()` | per-bad-IRET fix-up stack | `Trap::fixup_bad_iret` |
| `trap_init()` | per-boot IDT/IST setup | `Trap::trap_init` |
| `__warn_args()` / `WARN_trap` | per-WARN va_list synth | `Trap::warn_args` |

### compatibility contract

REQ-1: Trap-vector constants (`arch/x86/include/asm/traps.h`):
- X86_TRAP_DE = 0 (#DE, divide error, no errcode).
- X86_TRAP_DB = 1 (#DB, debug, no errcode, IST=4 on x86_64).
- X86_TRAP_NMI = 2 (#NMI, no errcode, IST=3 on x86_64; handled in nmi.c).
- X86_TRAP_BP = 3 (#BP, int3, no errcode).
- X86_TRAP_OF = 4 (#OF, INTO overflow, no errcode).
- X86_TRAP_BR = 5 (#BR, bound range, no errcode).
- X86_TRAP_UD = 6 (#UD, invalid opcode, no errcode).
- X86_TRAP_NM = 7 (#NM, device not available, no errcode).
- X86_TRAP_DF = 8 (#DF, double fault, errcode == 0, IST=1 on x86_64).
- X86_TRAP_OLD_MF = 9 (#MF, coprocessor segment overrun, legacy).
- X86_TRAP_TS = 10 (#TS, invalid TSS, errcode).
- X86_TRAP_NP = 11 (#NP, segment not present, errcode).
- X86_TRAP_SS = 12 (#SS, stack segment, errcode).
- X86_TRAP_GP = 13 (#GP, general protection, errcode).
- X86_TRAP_PF = 14 (#PF, page fault, errcode + CR2; handler in mm/fault.c).
- X86_TRAP_SPURIOUS = 15 (Pentium Pro erratum).
- X86_TRAP_MF = 16 (#MF, x87 FP, no errcode).
- X86_TRAP_AC = 17 (#AC, alignment check, errcode).
- X86_TRAP_MC = 18 (#MC, machine check, no errcode, IST=2 on x86_64; handler in mce/core.c).
- X86_TRAP_XF = 19 (#XF, SIMD FP, no errcode).
- X86_TRAP_VE = 20 (#VE, TDX virtualization exception, no errcode).
- X86_TRAP_CP = 21 (#CP, control-protection / CET, errcode; handler in cet.c).
- X86_TRAP_VC = 29 (#VC, SEV-ES VMM communication, errcode; handler in sev.c).
- X86_TRAP_IRET = 32 (x86_32 synthetic IRET-fault vector).

REQ-2: do_trap_no_signal(tsk, trapnr, str, regs, error_code):
- If v8086_mode(regs):
  - if trapnr < X86_TRAP_UD: handle_vm86_trap(... ); return 0 on consume.
- Else if !user_mode(regs):
  - if fixup_exception(regs, trapnr, error_code, 0): return 0 (kernel fix-up table consumed).
  - tsk.thread.error_code = error_code; tsk.thread.trap_nr = trapnr; die(str, regs, error_code) /* noreturn */.
- Else (user_mode):
  - if fixup_vdso_exception(regs, trapnr, error_code, 0): return 0 (vDSO fix-up consumed).
- Set tsk.thread.error_code = error_code; tsk.thread.trap_nr = trapnr; return -1 (signal must be delivered).

REQ-3: do_trap(trapnr, signr, str, regs, error_code, sicode, addr):
- If do_trap_no_signal returns 0: return (consumed).
- show_signal(tsk, signr, "trap ", str, regs, error_code) /* ratelimited */.
- If sicode == 0: force_sig(signr).
- Else: force_sig_fault(signr, sicode, addr).
- NOKPROBE_SYMBOL(do_trap).

REQ-4: do_error_trap(regs, error_code, str, trapnr, signr, sicode, addr):
- RCU_LOCKDEP_WARN(!rcu_is_watching(), ...).
- if notify_die(DIE_TRAP, str, regs, error_code, trapnr, signr) == NOTIFY_STOP: return.
- cond_local_irq_enable(regs); do_trap(...); cond_local_irq_disable(regs).

REQ-5: exc_divide_error (#DE):
- DEFINE_IDTENTRY(exc_divide_error) — no errcode.
- do_error_trap(regs, 0, "divide error", X86_TRAP_DE, SIGFPE, FPE_INTDIV, error_get_trap_addr(regs)).

REQ-6: exc_overflow (#OF, INTO):
- DEFINE_IDTENTRY(exc_overflow) — no errcode.
- do_error_trap(regs, 0, "overflow", X86_TRAP_OF, SIGSEGV, 0 /* no sicode */, NULL).

REQ-7: exc_bounds (#BR, BOUND):
- DEFINE_IDTENTRY(exc_bounds) — no errcode.
- notify_die(DIE_TRAP, "bounds", ...) NOTIFY_STOP → return.
- cond_local_irq_enable; if !user_mode(regs) → die("bounds", ...); else do_trap(X86_TRAP_BR, SIGSEGV, "bounds", ...); cond_local_irq_disable.

REQ-8: exc_invalid_op (#UD) — DEFINE_IDTENTRY_RAW, runs in raw entry context:
- If !user_mode(regs) ∧ handle_bug(regs): return (WARN/BUG/UBSan/CFI consumed in-place).
- Else: state = irqentry_enter(regs); instrumentation_begin; handle_invalid_op(regs) → SIGILL/ILL_ILLOPN; instrumentation_end; irqentry_exit.

REQ-9: handle_bug(regs) — noinstr:
- addr = regs->ip; ud_type = decode_bug(addr, &ud_imm, &ud_len).
- if ud_type == BUG_NONE: return false.
- instrumentation_begin; kmsan_unpoison_entry_regs(regs).
- /* Restore IF if set at fault */ — raw_local_irq_enable() if X86_EFLAGS_IF.
- Switch ud_type:
  - BUG_UD1_WARN: report_bug_entry((void *)pt_regs_val(regs, ud_imm), regs) — returns BUG_TRAP_TYPE_WARN if handled.
  - BUG_UD2: report_bug(regs->ip, regs) — return-as-WARN path; else fallthrough to CFI.
  - BUG_UDB | BUG_LOCK: handle_cfi_failure(regs) — FineIBT / LOCK-CFI consumed-as-WARN.
  - BUG_UD1_UBSAN: pr_crit(report_ubsan_failure(ud_imm), regs->ip) (if CONFIG_UBSAN_TRAP).
- If handled: advance regs->ip by ud_len; else: restore regs->ip = addr.
- raw_local_irq_disable() if X86_EFLAGS_IF. instrumentation_end. Return handled.

REQ-10: decode_bug(addr, *imm, *len):
- Strip Address-Size Override (`0x67`), `LOCK` (`0xf0`), and `REX` (`0x40..0x4f`) prefixes.
- If first opcode in 0x70..0x7f (Jcc.d8): require lock prefix → BUG_LOCK (CFI-LOCK).
- If 0xd6 (UDB): BUG_UDB.
- If `0x0f` (OPCODE_ESCAPE):
  - 2nd byte == SECOND_BYTE_OPCODE_UD2 (0x0b) → BUG_UD2.
  - 2nd byte == SECOND_BYTE_OPCODE_UD1 (0xb9) → decode ModRM/SIB/disp:
    - rm == 0 (mod 0/1/2 indirect via (%eax)): BUG_UD1_UBSAN with imm0/imm8/imm32.
    - mod == 0 ∧ rm == 2 (indirect via (%edx)): BUG_UD1_WARN with reg as pt_regs offset.
  - Else: BUG_NONE.
- record `*len = addr - start`.

REQ-11: exc_double_fault (#DF) — DEFINE_IDTENTRY_DF, runs on IST stack (IST index 1 on x86_64):
- On x86_64: error_code is 0 (architectural); IST stack via cpu_tss_rw.x86_tss.ist[1].
- ESPFIX64 fix-up: if (regs->sp >> P4D_SHIFT) == ESPFIX_PGD_ENTRY ∧ regs->cs == __KERNEL_CS ∧ regs->ip == native_irq_return_iret:
  - Synthesize a fake #GP(0) frame at TSS.sp0 with the failed IRET frame copied in.
  - Adjust regs->ip = asm_exc_general_protection; regs->sp = &gpregs->orig_ax; return.
- Otherwise: irqentry_nmi_enter; notify_die(DIE_TRAP, "double fault", ..., SIGSEGV).
- VMAP_STACK guard-page diagnosis: address = read_cr2(); if get_stack_guard_info(address, &info): handle_stack_overflow(regs, address, &info) — die("stack guard page") → panic.
- pr_emerg("PANIC: double fault, error_code: ..."); die("double fault", ...); panic("Machine halted.").

REQ-12: handle_stack_overflow(regs, fault_address, info) — VMAP_STACK only, noreturn:
- printk(KERN_EMERG, "BUG: %s stack guard page was hit at %px (stack is %px..%px)").
- die("stack guard page", regs, 0); panic("%s stack guard hit", name).

REQ-13: exc_invalid_tss (#TS):
- DEFINE_IDTENTRY_ERRORCODE — errcode = selector.
- do_error_trap(regs, error_code, "invalid TSS", X86_TRAP_TS, SIGSEGV, 0, NULL).

REQ-14: exc_segment_not_present (#NP):
- DEFINE_IDTENTRY_ERRORCODE.
- do_error_trap(regs, error_code, "segment not present", X86_TRAP_NP, SIGBUS, 0, NULL).

REQ-15: exc_stack_segment (#SS):
- DEFINE_IDTENTRY_ERRORCODE.
- do_error_trap(regs, error_code, "stack segment", X86_TRAP_SS, SIGBUS, 0, NULL).

REQ-16: exc_alignment_check (#AC):
- DEFINE_IDTENTRY_ERRORCODE — errcode = 0 architecturally, may carry split-lock indicator.
- notify_die(DIE_TRAP, "alignment check", ..., X86_TRAP_AC, SIGBUS) == NOTIFY_STOP → return.
- If !user_mode(regs): die("Split lock detected\n", regs, error_code).
- local_irq_enable.
- handle_user_split_lock(regs, error_code): consume and go to out.
- Else do_trap(X86_TRAP_AC, SIGBUS, "alignment check", regs, error_code, BUS_ADRALN, NULL).
- out: local_irq_disable.

REQ-17: exc_general_protection (#GP):
- DEFINE_IDTENTRY_ERRORCODE — errcode = segment selector or 0.
- If user_mode(regs) ∧ try_fixup_enqcmd_gp(): return /* PASID MSR populated; retry */.
- cond_local_irq_enable.
- If v8086_mode(regs): handle_vm86_fault((kernel_vm86_regs *)regs, error_code); return.
- If user_mode(regs):
  - fixup_iopl_exception (CLI/STI NOP-as-IOPL emulation) → exit.
  - fixup_vdso_exception(regs, X86_TRAP_GP, error_code, 0) → exit.
  - fixup_umip_exception → exit (UMIP SLDT/STR/SIDT/SGDT/SMSW emulation).
  - emulate_vsyscall_gp → exit.
  - Else: gp_user_force_sig_segv(regs, X86_TRAP_GP, error_code, GPFSTR) → exit.
- Kernel-mode: gp_try_fixup_and_notify(regs, X86_TRAP_GP, error_code, desc, 0) → exit.
- Compose descriptive string:
  - If error_code != 0: "segment-related " GPFSTR.
  - Else hint = get_kernel_gp_address(regs, &gp_addr); decode "non-canonical", "maybe for", "LASS violation for", "NULL pointer" hints.
- die_addr(desc, regs, error_code, hint == GP_NON_CANONICAL ? gp_addr : 0).

REQ-18: try_fixup_enqcmd_gp() — CONFIG_ARCH_HAS_CPU_PASID:
- lockdep_assert_irqs_disabled.
- if !cpu_feature_enabled(X86_FEATURE_ENQCMD): return false.
- if !mm_valid_pasid(current->mm): return false.
- pasid = mm_get_enqcmd_pasid(current->mm).
- if current->pasid_activated: return false /* must be a different #GP cause */.
- wrmsrq(MSR_IA32_PASID, pasid | MSR_IA32_PASID_VALID); current->pasid_activated = 1; return true.

REQ-19: get_kernel_gp_address(regs, *addr) — hint enum:
- copy_from_kernel_nofault MAX_INSN_SIZE bytes at regs->ip → GP_NO_HINT on failure.
- insn_decode_kernel(&insn, insn_buf) — GP_NO_HINT on failure.
- *addr = insn_get_addr_ref(&insn, regs); if *addr == -1UL: GP_NO_HINT.
- On x86_64:
  - *addr >= ~__VIRTUAL_MASK → GP_CANONICAL.
  - *addr + insn.opnd_bytes - 1 > __VIRTUAL_MASK → GP_NON_CANONICAL.
  - *addr < PAGE_SIZE → GP_NULL_POINTER.
  - X86_FEATURE_LASS enabled → GP_LASS_VIOLATION.
- Else GP_CANONICAL.

REQ-20: exc_int3 (#BP) — DEFINE_IDTENTRY_RAW:
- smp_text_poke_int3_handler(regs) consumes text-poke INT3 instantly without irqentry; return.
- If user_mode(regs): irqentry_enter_from_user_mode; do_int3_user → notify_die(DIE_INT3) / SIGTRAP via do_trap; irqentry_exit_to_user_mode.
- Else: irqentry_nmi_enter; if !do_int3(regs) → die("int3", regs, 0); irqentry_nmi_exit.
- do_int3: KGDB_LL_TRAP → kprobe_int3_handler → notify_die(DIE_INT3) === NOTIFY_STOP. NOKPROBE.

REQ-21: exc_debug (#DB) — split entry points:
- DEFINE_IDTENTRY_DEBUG(exc_debug) — IST stack entry, calls exc_debug_kernel(regs, debug_read_reset_dr6()).
- DEFINE_IDTENTRY_DEBUG_USER(exc_debug) — user-mode entry, regular task stack, calls exc_debug_user(regs, debug_read_reset_dr6()).
- DEFINE_FREDENTRY_DEBUG(exc_debug) — FRED-only: ring 0 on dedicated #DB stack, ring 3 on current task stack; user_mode(regs) dispatches between exc_debug_user / exc_debug_kernel.
- 32-bit: DEFINE_IDTENTRY_RAW(exc_debug) — common entry; dispatches by user_mode(regs).
- IST stack index for #DB on x86_64 = 4.

REQ-22: debug_read_reset_dr6():
- get_debugreg(dr6, 6); dr6 ^= DR6_RESERVED /* flip polarity */.
- set_debugreg(DR6_RESERVED, 6) /* clear to architectural reset 0xFFFF0FF0 */.
- Return dr6.

REQ-23: exc_debug_kernel(regs, dr6) — noinstr:
- dr7 = local_db_save() /* disable HW breakpoints during handler */.
- irq_state = irqentry_nmi_enter(regs).
- instrumentation_begin.
- WARN_ON_ONCE(user_mode(regs)).
- If TIF_BLOCKSTEP: re-arm MSR_IA32_DEBUGCTLMSR DEBUGCTLMSR_BTF (SDM says #DB clears BTF, but ptrace requested it).
- If !FRED ∧ (dr6 & DR_STEP) ∧ is_sysenter_singlestep(regs): clear DR_STEP (spurious SYSENTER TF).
- If !dr6: goto out.
- If notify_debug(regs, &dr6) /* KGDB / HW breakpoint / kprobe consumed */: goto out.
- WARN_ON_ONCE(dr6 & DR_STEP) /* kernel-mode TF outside kprobes/KGDB is wonky */; clear regs->flags X86_EFLAGS_TF.
- out: instrumentation_end; irqentry_nmi_exit(regs, irq_state); local_db_restore(dr7).

REQ-24: exc_debug_user(regs, dr6) — noinstr:
- WARN_ON_ONCE(!user_mode(regs)).
- irqentry_enter_from_user_mode(regs); instrumentation_begin.
- current->thread.virtual_dr6 = dr6 & DR_STEP /* ptrace_triggered will OR in DR_TRAPn */.
- clear_thread_flag(TIF_BLOCKSTEP) /* HW cleared BTF */.
- icebp = !dr6 /* dr6==0 → icebp/int01 software-generated */.
- if notify_debug(regs, &dr6): goto out.
- local_irq_enable.
- if v8086_mode(regs): handle_vm86_trap((kernel_vm86_regs *)regs, 0, X86_TRAP_DB); goto out_irq.
- if dr6 & DR_BUS_LOCK: handle_bus_lock(regs) /* split-lock bus-lock */.
- dr6 |= current->thread.virtual_dr6.
- if dr6 & (DR_STEP | DR_TRAP_BITS) ∨ icebp: send_sigtrap(regs, 0, get_si_code(dr6)).
- out_irq: local_irq_disable; out: instrumentation_end; irqentry_exit_to_user_mode.

REQ-25: notify_debug(regs, *dr6):
- notify_die(DIE_DEBUG, "debug", regs, (long)dr6, 0, SIGTRAP) == NOTIFY_STOP → return true.

REQ-26: is_sysenter_singlestep(regs):
- x86_32: regs->ip ∈ [__begin_SYSENTER_singlestep_region, __end_SYSENTER_singlestep_region).
- CONFIG_IA32_EMULATION: regs->ip ∈ [entry_SYSENTER_compat, __end_entry_SYSENTER_compat).
- 64-bit native: false.

REQ-27: exc_coprocessor_error (#MF):
- DEFINE_IDTENTRY — no errcode.
- math_error(regs, X86_TRAP_MF) → si_code via fpu__exception_code(fpu, X86_TRAP_MF); SIGFPE.

REQ-28: exc_simd_coprocessor_error (#XF):
- DEFINE_IDTENTRY — no errcode.
- CONFIG_X86_INVD_BUG: if !X86_FEATURE_XMM (AMD 486 INVD-in-CPL0): __exc_general_protection(regs, 0); return.
- math_error(regs, X86_TRAP_XF) → SIGFPE.

REQ-29: math_error(regs, trapnr):
- cond_local_irq_enable.
- If !user_mode(regs):
  - if fixup_exception(regs, trapnr, 0, 0): exit.
  - notify_die(DIE_TRAP, str, ..., SIGFPE) != NOTIFY_STOP → die(str, ...).
- Else:
  - fpu_sync_fpstate(fpu) /* fault-time FP register state visible to handler */.
  - si_code = fpu__exception_code(fpu, trapnr); !si_code → spurious, exit.
  - fixup_vdso_exception(regs, trapnr, 0, 0) → exit.
  - force_sig_fault(SIGFPE, si_code, uprobe_get_trap_addr(regs)).
- exit: cond_local_irq_disable.

REQ-30: exc_device_not_available (#NM):
- DEFINE_IDTENTRY — no errcode.
- cr0 = read_cr0().
- handle_xfd_event(regs) /* x86_64 + X86_FEATURE_XFD */ → return on consume.
- CONFIG_MATH_EMULATION: if !X86_FEATURE_FPU ∧ (cr0 & X86_CR0_EM): cond_local_irq_enable; math_emulate(&info); cond_local_irq_disable; return.
- WARN(cr0 & X86_CR0_TS): clear CR0.TS and continue (transient lazy-FPU state — should not happen post-eagerfpu).
- Else die("unexpected #NM exception", regs, 0).

REQ-31: handle_xfd_event(regs):
- !CONFIG_X86_64 ∨ !X86_FEATURE_XFD → return false.
- rdmsrq(MSR_IA32_XFD_ERR, xfd_err); !xfd_err → return false.
- wrmsrq(MSR_IA32_XFD_ERR, 0) /* must clear or hang */.
- WARN(!user_mode(regs)) → die path.
- local_irq_enable; err = xfd_enable_feature(xfd_err).
- err == -EPERM → force_sig_fault(SIGILL, ILL_ILLOPC, error_get_trap_addr).
- err == -EFAULT → force_sig(SIGSEGV).
- local_irq_disable; return true.

REQ-32: exc_spurious_interrupt_bug — Pentium Pro erratum 15:
- DEFINE_IDTENTRY — silent NOP handler (vector 15 may fire on APIC mixed-mode + Virtual Wire).

REQ-33: exc_virtualization_exception (#VE) — CONFIG_INTEL_TDX_GUEST:
- DEFINE_IDTENTRY — no errcode (vector 20).
- tdx_get_ve_info(&ve) /* mandatory TDGETVEINFO — until called, all interrupts including NMIs are blocked at the TDX module */.
- cond_local_irq_enable.
- if !tdx_handle_virt_exception(regs, &ve): ve_raise_fault(regs, 0, ve.gla) /* treat as #GP(0) */.
- cond_local_irq_disable.

REQ-34: ve_raise_fault(regs, error_code, address):
- user_mode(regs): gp_user_force_sig_segv(regs, X86_TRAP_VE, error_code, "VE fault"); return.
- gp_try_fixup_and_notify(regs, X86_TRAP_VE, error_code, "VE fault", address): return.
- die_addr("VE fault", regs, error_code, address).

REQ-35: x86_32-only iret_error:
- DEFINE_IDTENTRY_SW(iret_error) — software vector X86_TRAP_IRET.
- local_irq_enable; notify_die(DIE_TRAP, "iret exception", X86_TRAP_IRET, SIGILL) != NOTIFY_STOP → do_trap(...SIGILL, "iret exception", ILL_BADSTK, NULL).

REQ-36: sync_regs(eregs) — CONFIG_X86_64:
- Return ((struct pt_regs *)current_top_of_stack()) - 1; if different from eregs, copy.
- Asmlinkage, noinstr — used by entry_64.S to switch off IST stack to thread stack.

REQ-37: vc_switch_off_ist(regs) — CONFIG_AMD_MEM_ENCRYPT:
- if ip_within_syscall_gap(regs): sp = current_top_of_stack(); else use regs->sp.
- get_stack_info_noinstr → if STACK_TYPE_ENTRY ∨ type > STACK_TYPE_EXCEPTION_LAST: sp = __this_cpu_ist_top_va(VC2) /* fallback VC2 IST */.
- ALIGN_DOWN sp by 8, subtract sizeof(pt_regs); copy regs into new slot; return.

REQ-38: fixup_bad_iret(bad_regs) — CONFIG_X86_64:
- new_stack = ((cpu_tss_rw.x86_tss.sp0) - sizeof(pt_regs)).
- __memcpy(&tmp.ip, bad_regs->sp, 5*8) /* IRET frame from current sp */.
- __memcpy(&tmp, bad_regs, offsetof(pt_regs, ip)) /* rest of regs from current */.
- __memcpy(new_stack, &tmp, sizeof(tmp)).
- BUG_ON(!user_mode(new_stack)); return new_stack.

REQ-39: trap_init():
- setup_cpu_entry_areas() /* per-cpu entry area before IST */.
- sev_es_init_vc_handling() /* GHCB pages */.
- cpu_init_exception_handling(true) /* TSS + IST stack pointers */.
- if !X86_FEATURE_FRED: idt_setup_traps() /* install IDT vectors */.
- cpu_init() /* finish bringup */.

REQ-40: Per-IST stack assignments (x86_64, arch/x86/include/asm/cpu_entry_area.h):
- IST 1 (DF) — exc_double_fault.
- IST 2 (NMI) — exc_nmi (in nmi.c).
- IST 3 (MCE) — exc_machine_check (in mce/core.c).
- IST 4 (DB) — exc_debug.
- IST 5 (VC) — exc_vmm_communication (in sev.c, with VC2 fallback for nested).

REQ-41: CET / #CP — separate file `arch/x86/kernel/cet.c`:
- exc_control_protection(regs, error_code) handles shadow-stack / IBT failures.
- error_code bit-15 = endbranch-violation; lower bits encode #CP sub-cause.
- traps.c does not own this handler but covers `X86_TRAP_CP` vector reservation; the IDT install is done in `idt_setup_traps()`.

REQ-42: __warn_args(args, regs) — CONFIG_HAVE_ARCH_BUG_FORMAT_ARGS:
- Build va_list from pt_regs RDI..R9 in args.regs[0..5].
- gp_offset = 1*8 (skip first arg); fp_offset = 6*8 + 8*16; reg_save_area = &args.regs; overflow_arg_area = regs->sp.
- If regs->ip == &__WARN_trap: overflow_arg_area += 8 (skip return address).
- Returns &args.args (va_list).

REQ-43: Notifier chains:
- DIE_TRAP — for #DE/#OF/#BR/#TS/#NP/#SS/#AC/#DF.
- DIE_GPF — for #GP fix-up notify.
- DIE_INT3 — for #BP (kprobes/KGDB).
- DIE_DEBUG — for #DB (kprobes/KGDB/HW breakpoints/uprobe).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `do_trap_no_signal_kernel_die_or_fixup` | INVARIANT | per-do_trap_no_signal: !user_mode ⟹ fixup_exception consumed ∨ die() (noreturn). |
| `bug_decode_ip_advance_iff_handled` | INVARIANT | per-handle_bug: `regs.ip += ud_len` only when handled; otherwise restored to original. |
| `double_fault_runs_on_ist1` | INVARIANT | per-exc_double_fault: entry stack ∈ IST[1] range; exits via `die()` ∨ `panic()` ∨ synthesized #GP. |
| `debug_kernel_dr7_balanced` | INVARIANT | per-exc_debug_kernel: `local_db_save()` paired with `local_db_restore(dr7)` on all paths. |
| `debug_dr6_reset_after_read` | INVARIANT | per-debug_read_reset_dr6: DR6 written with DR6_RESERVED before handler proceeds. |
| `gp_enqcmd_pasid_only_once` | INVARIANT | per-try_fixup_enqcmd_gp: `current->pasid_activated == 0` precondition; sets to 1 on success. |
| `xfd_err_cleared_before_enable` | INVARIANT | per-handle_xfd_event: `wrmsrq(MSR_IA32_XFD_ERR, 0)` issued before `xfd_enable_feature`. |
| `int3_text_poke_first` | INVARIANT | per-exc_int3: `smp_text_poke_int3_handler` runs before any irqentry_*. |
| `ve_get_info_before_irq` | INVARIANT | per-exc_virtualization_exception: `tdx_get_ve_info` called with interrupts still blocked at TDX module. |

### Layer 2: TLA+

`arch/x86/traps.tla`:
- Per-vector entry → per-irqentry (user vs kernel vs nmi) → per-fixup → per-signal/die → per-return.
- Per-IST stack invariant (#DF/#NMI/#MC/#DB/#VC).
- Per-text-poke + per-int3 ordering.
- Properties:
  - `safety_user_signals_delivered_or_no_signal` — per-do_trap: returns either via fixup-consume or signal-deliver path.
  - `safety_kernel_unrecoverable_dies` — per-do_trap_no_signal: !user_mode ∧ !fixup ⟹ die() noreturn.
  - `safety_df_no_return_unless_espfix_fixup` — per-#DF: either ESPFIX synth-return or panic.
  - `safety_db_ist_recursion_bound` — per-#DB: nested #DB cannot recurse beyond `local_db_save` masking window.
  - `safety_int3_text_poke_atomic` — per-#BP: text-poke consume path never calls instrumentable code.
  - `safety_enqcmd_pasid_idempotent` — per-#GP fix-up: PASID is written at most once per task per gen.
  - `liveness_per_trap_terminates_or_signals` — per-exc_*: handler reaches return or oops in bounded steps.
  - `safety_ve_blocks_until_tdgetveinfo` — per-#VE: all interrupts blocked until tdx_get_ve_info returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Trap::do_trap_no_signal` post: ret ∈ {0, -1} ∨ die() | `Trap::do_trap_no_signal` |
| `Trap::do_trap` post: signal delivered iff `do_trap_no_signal` returned -1 | `Trap::do_trap` |
| `Trap::handle_bug` post: return ⟹ regs.ip advanced; !return ⟹ regs.ip preserved | `Trap::handle_bug` |
| `Trap::decode_bug` post: BUG_UD2/UDB/LOCK have well-defined `*len`; BUG_UD1_* have `*imm` set | `Trap::decode_bug` |
| `Trap::exc_double_fault` post: returns iff espfix-fixup path; else noreturn | `Trap::exc_double_fault` |
| `Trap::exc_debug_kernel` post: dr7 restored on all exits | `Trap::exc_debug_kernel` |
| `Trap::exc_debug_user` post: SIGTRAP delivered iff dr6 has step/trap-bits ∨ icebp | `Trap::exc_debug_user` |
| `Trap::exc_general_protection` post: user-mode path delivers SIGSEGV unless fix-up consumed; kernel-mode path dies unless fix-up | `Trap::exc_general_protection` |
| `Trap::exc_int3` post: NOKPROBE on `do_int3`; text-poke handler bypasses irqentry | `Trap::exc_int3` |
| `Trap::exc_alignment_check` post: kernel-mode dies on Split-lock; user-mode either consumed by handle_user_split_lock or SIGBUS | `Trap::exc_alignment_check` |
| `Trap::trap_init` post: cpu_entry_area set up before IDT install | `Trap::trap_init` |

### Layer 4: Verus/Creusot functional

`Per-CPU exception → per-IDT dispatch → per-IST switch (#DF/#DB/#MC/#NMI/#VC) → per-handler → per-signal-or-die` semantic equivalence: per Intel SDM Vol 3 § 6.15 (Interrupts and Exceptions), § 7.7 (IST Mechanism), and § 17.3 (CET).

### hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

Trap-handler reinforcement:

- **Per-IST stack for #DF/#NMI/#MC/#DB/#VC** — defense against per-faulting-stack-causes-double-fault chain (each critical vector has its own pre-allocated stack so SS:SP corruption cannot escalate).
- **Per-#DF ESPFIX64 #GP synthesis** — defense against per-IRET-to-bad-SS triple-fault on 64-bit kernels accessed via 16-bit SS segments.
- **Per-VMAP_STACK guard-page #DF diagnosis** — defense against per-silent-stack-overflow corrupting random memory; we panic loudly with the address range.
- **Per-`local_db_save` around #DB kernel handler** — defense against per-HW-breakpoint-while-handling-#DB infinite recursion.
- **Per-DR6 reset to DR6_RESERVED (0xFFFF0FF0)** — defense against per-stale DR6-bits leaking into next #DB.
- **Per-`smp_text_poke_int3_handler` runs before irqentry_enter** — defense against per-irqentry-itself trapping in INT3 during live patching, recursive WARNs.
- **Per-#UD `decode_bug` strict prefix handling (ASOP/LOCK/REX)** — defense against per-malicious `0f 0b` lookalike injecting unintended WARN imm.
- **Per-#GP ENQCMD PASID activation gated on `mm_valid_pasid` + `!current->pasid_activated`** — defense against per-spurious-PASID-write or per-PASID-confusion.
- **Per-#GP get_kernel_gp_address bounded by `copy_from_kernel_nofault`** — defense against per-handler dereferencing a fault-causing RIP.
- **Per-#GP LASS hint kept distinct from non-canonical hint** — defense against per-mis-attribution that confuses oops triage.
- **Per-#AC kernel-mode dies on Split-lock** — defense against per-kernel splitting AC bus locks that signal CPU bus contention.
- **Per-#NM `MSR_IA32_XFD_ERR` cleared before `xfd_enable_feature`** — defense against per-stuck-XFD looping faults.
- **Per-#VE `tdx_get_ve_info` first (before IRQ enable)** — defense against per-nested-#VE corruption of VE info window.
- **Per-`fixup_vdso_exception` for user faults** — defense against per-vDSO-instruction-causing-fault that should be NOP'd silently (UMIP / vsyscall / IOPL emulation).
- **Per-WARN/BUG `__WARN_trap` static-call distinct from generic UD2** — defense against per-misclassification of UBSan / FineIBT / static_call patches as ordinary illegal-instruction.
- **Per-`trap_init` ordering (cpu_entry_area → IST → IDT)** — defense against per-IDT-fires-before-IST-stacks-allocated boot-time crash.

