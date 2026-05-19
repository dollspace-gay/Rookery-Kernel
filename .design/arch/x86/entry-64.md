# Tier-3: arch/x86/entry/entry_64.S + entry_64_compat.S — x86_64 hardware-to-software boundary

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: arch/x86/00-overview.md
upstream-paths:
  - arch/x86/entry/entry_64.S (~1570 lines)
  - arch/x86/entry/entry_64_compat.S (~299 lines)
  - arch/x86/entry/syscall_64.c (do_syscall_64)
  - arch/x86/entry/syscall_32.c (do_int80_emulation, do_fast_syscall_32)
  - arch/x86/entry/calling.h (SAVE_AND_SWITCH_TO_KERNEL_CR3 / SWITCH_TO_USER_CR3_*)
  - arch/x86/include/asm/cpu_entry_area.h
  - arch/x86/kernel/nmi.c
-->

## Summary

The lowest-level x86_64 boundary between hardware-delivered control transfers (SYSCALL, INT 0x80, IRQ, exception, NMI, MCE, SYSENTER-compat) and the C kernel. Per-CPU GSBASE swap (`swapgs`), per-CPU CR3 swap for KPTI (Kernel Page Table Isolation / PTI), per-CPU `cpu_entry_area` trampoline stack, per-NMI nesting state machine (`first_nmi` / `repeat_nmi` / `end_repeat_nmi`), per-paranoid-entry pre-GS-validation path. Every syscall begins here; every IRQ vector lands here; every #PF / #GP / #DB / #BP / #UD / #DF / #MC enters here. Critical for: ABI fidelity (syscall calling convention), Meltdown mitigation (KPTI), NMI re-entry correctness, SMAP/SMEP enforcement, IST stack discipline for #DB/#NMI/#DF/#MC.

This Tier-3 covers `entry_64.S` (~1570 lines) plus the 32-bit compatibility entry siblings in `entry_64_compat.S` (~299 lines); the C dispatch into `do_syscall_64` / `do_int80_emulation` / `do_fast_syscall_32` is described as the immediate downstream boundary.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `entry_SYSCALL_64` | per-CPU SYSCALL MSR_LSTAR target | `arch::x86::entry::syscall_64` |
| `entry_SYSCALL_64_safe_stack` | per-CPU label after stack switch | `arch::x86::entry::syscall_64_safe_stack` |
| `entry_SYSCALL_64_after_hwframe` | per-CPU label after pt_regs build | `arch::x86::entry::syscall_64_after_hwframe` |
| `entry_SYSRETQ_unsafe_stack` | per-CPU SYSRETQ-prep label | `arch::x86::entry::sysretq_unsafe_stack` |
| `entry_SYSRETQ_end` | per-CPU SYSRETQ-end label | `arch::x86::entry::sysretq_end` |
| `__switch_to_asm` | per-task low-level context switch | `arch::x86::entry::switch_to_asm` |
| `ret_from_fork_asm` | per-fork first-return path | `arch::x86::entry::ret_from_fork_asm` |
| `common_interrupt_return` | per-IRQ return tail | `arch::x86::entry::common_interrupt_return` |
| `swapgs_restore_regs_and_return_to_usermode` | per-return-to-user PTI/swapgs tail | `arch::x86::entry::swapgs_restore_to_user` |
| `restore_regs_and_return_to_kernel` | per-return-to-kernel tail | `arch::x86::entry::restore_to_kernel` |
| `native_irq_return_iret` | per-IRET instruction site (for #GP fixup) | `arch::x86::entry::native_irq_return_iret` |
| `asm_load_gs_index` | per-load_gs_index swapgs-recovery | `arch::x86::entry::asm_load_gs_index` |
| `paranoid_entry` | per-paranoid (NMI/MCE/#DB) entry: read MSR_GS_BASE rather than blind swapgs | `arch::x86::entry::paranoid_entry` |
| `paranoid_exit` | per-paranoid exit | `arch::x86::entry::paranoid_exit` |
| `error_entry` | per-non-paranoid exception entry | `arch::x86::entry::error_entry` |
| `error_return` | per-error exit | `arch::x86::entry::error_return` |
| `asm_exc_nmi` | per-NMI vector + nesting state machine | `arch::x86::entry::asm_exc_nmi` |
| `first_nmi` / `repeat_nmi` / `end_repeat_nmi` | per-NMI nesting trampoline | `arch::x86::entry::nmi::{first,repeat,end_repeat}_nmi` |
| `entry_SYSENTER_compat` | per-CPU MSR_IA32_SYSENTER_EIP target (32-bit) | `arch::x86::entry::sysenter_compat` |
| `entry_SYSCALL_compat` | per-CPU MSR_CSTAR target (32-bit fast syscall) | `arch::x86::entry::syscall_compat` |
| `entry_SYSRETL_compat_unsafe_stack` | per-CPU SYSRETL-prep label (compat) | `arch::x86::entry::sysretl_compat_unsafe_stack` |
| `int80_emulation` | per-INT-0x80 legacy 32-bit syscall stub | `arch::x86::entry::int80_emulation` |
| `do_syscall_64` (C) | per-syscall C dispatch (x64) | `arch::x86::syscall::do_syscall_64` |
| `do_int80_emulation` (C) | per-INT-0x80 C dispatch | `arch::x86::syscall::do_int80_emulation` |
| `do_fast_syscall_32` (C) | per-fast-syscall-32 C dispatch | `arch::x86::syscall::do_fast_syscall_32` |
| `SAVE_AND_SWITCH_TO_KERNEL_CR3` (calling.h) | per-entry KPTI CR3 swap | `arch::x86::entry::save_and_switch_to_kernel_cr3!` (macro) |
| `SWITCH_TO_USER_CR3_STACK` / `_NOSTACK` (calling.h) | per-return KPTI CR3 swap | `arch::x86::entry::switch_to_user_cr3_*!` (macros) |

## Compatibility contract

REQ-1: SYSCALL fast-path (`entry_SYSCALL_64` at MSR_LSTAR):
- Hardware on SYSCALL: RIP←MSR_LSTAR; RFLAGS←(RFLAGS & ~MSR_FMASK); CS←(MSR_STAR>>32) & 0xFFFC; SS←CS+8; RCX←RIP_user; R11←RFLAGS_user; no stack switch by HW (RSP is still user RSP).
- swapgs (HW GS=user → KERNEL_GS_BASE=cpu_entry_area).
- SWITCH_TO_KERNEL_CR3 (KPTI): write kernel CR3 from `pcpu_hot.user_pcid_flush_mask`-derived scratch (or no-op if !X86_FEATURE_PTI).
- Stash user RSP in `cpu_tss_rw.x86_tss.sp2` (per-CPU scratch slot).
- Load kernel RSP from `pcpu_hot.top_of_stack`.
- Build a synthetic interrupt frame on the kernel stack: push $__USER_DS (SS), push user RSP, push R11 (RFLAGS), push $__USER_CS (CS), push RCX (RIP), push $-1 (orig_ax).
- PUSH_AND_CLEAR_REGS rax=$-ENOSYS — pushes all GPRs into `struct pt_regs` and pre-fills RAX with -ENOSYS (so unknown syscall returns -ENOSYS).
- mov %rsp, %rdi; mov %rax, %rsi; call do_syscall_64.

REQ-2: SYSRET fast-return (`entry_SYSRETQ_unsafe_stack`):
- POP_REGS pop_rdi=0 (RDI restored separately because we use it for CR3 scratch).
- SWITCH_TO_USER_CR3_STACK scratch_reg=%rdi.
- pop %rdi.
- pop %rsp (back to user RSP).
- swapgs.
- sysretq (HW: RIP←RCX; RFLAGS←R11; CS←(MSR_STAR>>48)+16; SS←CS+8).

REQ-3: SYSRET-vs-IRET decision in `do_syscall_64`:
- SYSRET requires: RIP canonical, RIP == RCX_user, RCX_user == RIP, R11 == RFLAGS, RFLAGS_user has TF/IF state expressible by RFMASK side-effect, RSP user has no extra bits, no work in TIF_NEED_RESCHED / TIF_SIGPENDING.
- If any test fails: fall through to `swapgs_restore_regs_and_return_to_usermode` (IRET slow path).
- Cited rationale: SYSRET silently corrupts on non-canonical RCX (CVE-2014-9322 class); see comment at `entry_SYSCALL_64` regarding RCX validation.

REQ-4: KPTI (Kernel Page Table Isolation / PTI) CR3 swap discipline:
- On entry: SAVE_AND_SWITCH_TO_KERNEL_CR3 (per-CPU `cpu_tlbstate.user_pcid_flush_mask`-aware).
- On exit: SWITCH_TO_USER_CR3 (or `_STACK` / `_NOSTACK` variants based on stack reachability).
- Must happen before any kernel data dereference and after any user data dereference; if X86_FEATURE_PTI absent, both macros NOP.

REQ-5: swapgs ordering invariant:
- On entry from user: swapgs FIRST, then SWITCH_TO_KERNEL_CR3 — because CR3-switch uses GS-relative `pcpu_hot` access.
- On exit to user: SWITCH_TO_USER_CR3 FIRST, then swapgs.
- Reverse on `paranoid_entry`: see REQ-12.

REQ-6: `entry_SYSCALL_64_after_hwframe` is a stable global label for tools (perf, kprobes) that need the post-pt_regs PC; ABI-stable.

REQ-7: `common_interrupt_return` is the unified IRQ/exception return tail; two branches:
- `swapgs_restore_regs_and_return_to_usermode` — back to user (with PTI + swapgs).
- `restore_regs_and_return_to_kernel` — back to kernel (no swapgs, no CR3 swap).
- Decision by inspecting saved CS on the stack (CS_user vs CS_kernel).

REQ-8: `error_entry` (non-paranoid exception path):
- Test saved CS: if from user → swapgs + SWITCH_TO_KERNEL_CR3 + fall into normal C handler.
- If from kernel and PC is in the SYSCALL gap (between `entry_SYSCALL_64` and `entry_SYSCALL_64_safe_stack`) → treat as user-mode and do swapgs anyway (because GS is still user-GS at that PC).
- Otherwise → no swapgs, no CR3 swap (already kernel).

REQ-9: `paranoid_entry` (NMI, #MC, #DB, #DF — paths that can interrupt themselves or arrive when GS is unknown):
- Read MSR_GS_BASE (rdmsr). If high bit set → already kernel-GS, skip swapgs; otherwise swapgs.
- SAVE_AND_SWITCH_TO_KERNEL_CR3 with explicit `save_reg=%r14` (so `paranoid_exit` can restore the prior PCID flush mask state).
- This avoids the swapgs-twice bug class where a nested NMI between swapgs and stack-switch would corrupt state.

REQ-10: `paranoid_exit`:
- Restore saved CR3 from R14 (the `save_reg` slot).
- If pre-entry GS was user → swapgs; else skip.
- Strictly inverse of `paranoid_entry`.

REQ-11: NMI nesting state machine (`asm_exc_nmi`):
- HW: NMIs masked while handler runs; un-masked on first IRET.
- Problem: handler may execute IRET via `paranoid_exit` for the synthetic frame, which un-masks NMI; a new NMI can then arrive in the middle of completing the first; without the trampoline this would lose the first NMI's context.
- Solution: nesting state via three labels:
  - `first_nmi` — initial entry; copies the HW NMI frame onto a per-CPU `nmi_stack`; sets up "outermost" frame.
  - `repeat_nmi` — re-entrant entry after `first_nmi` has set up its frame; HW IRET sent here loops back to drain pending NMI.
  - `end_repeat_nmi` — sentinel ; if RIP is in [`repeat_nmi`, `end_repeat_nmi`) at NMI entry, treat as nested.
- Per-CPU `nmi_executing` flag in cpu_entry_area; nested NMI just modifies the outer frame's iret target to `repeat_nmi` and returns; outer NMI sees flag and loops.

REQ-12: `asm_exc_nmi` entry sequence:
- If from kernel (CS_kernel): jump to `.Lnmi_from_kernel` — uses paranoid_entry semantics (rdmsr GS).
- If from user: swapgs + SWITCH_TO_KERNEL_CR3 + call exc_nmi + jump to `swapgs_restore_regs_and_return_to_usermode`.

REQ-13: `int80_emulation` (legacy INT 0x80):
- HW delivers via IDT vector 0x80 with `DPL=3` (user can invoke).
- ASM stub `int80_emulation` immediately tail-jumps to C `do_int80_emulation`.
- C side: forces 32-bit syscall ABI (truncates RAX to EAX index, fetches 32-bit args from EBX, ECX, EDX, ESI, EDI, EBP); calls `sys_call_table[nr]`.
- Available regardless of `CONFIG_IA32_EMULATION` (kept for "echo 0 >/proc/sys/abi/ia32_emulation" — see comments in `syscall_32.c`).

REQ-14: `entry_SYSENTER_compat` (Intel 32-bit fast syscall):
- Hardware: SS/SP fixed by MSR_IA32_SYSENTER_*; CS/EIP from MSRs; no register save.
- swapgs; SWITCH_TO_KERNEL_CR3; build pt_regs (with synthetic SS/RSP/CS/EIP for IRET compat); fix EFLAGS.IF (which HW clears); jump into common compat C path.
- `.Lsysenter_fix_flags` patches RFLAGS to set IF before C entry.

REQ-15: `entry_SYSCALL_compat` (AMD 32-bit fast syscall via MSR_CSTAR):
- Mirrors REQ-1 but for 32-bit. Stack and reg conventions per i386 syscall ABI.
- Returns via `entry_SYSRETL_compat_unsafe_stack` → SWITCH_TO_USER_CR3_NOSTACK → swapgs → sysretl.

REQ-16: `__switch_to_asm`:
- per-task low-level switch: save callee-saved regs (RBP, RBX, R12-R15) on prev's stack, swap RSP, restore from next's stack, jump to `__switch_to` (C).
- Used by `context_switch()` after preparing both task_structs.

REQ-17: `ret_from_fork_asm`:
- First return path for new fork()ed kernel threads / userspace forks.
- For kthreads: calls `kthread_frame_init`'d fn(arg).
- For userspace: tail-jumps to `swapgs_restore_regs_and_return_to_usermode` to return to user RIP saved in child's pt_regs.

REQ-18: `asm_load_gs_index`:
- Wrapper around `mov %ax, %gs` that catches #GP via exception table, restores `__USER_DS=0` on fault, and re-issues swapgs to recover; used by `load_gs_index` C helper.

## Acceptance Criteria

- [ ] AC-1: SYSCALL from user: hardware delivers RIP=MSR_LSTAR; software does swapgs → CR3 swap → stack switch → pt_regs build → `do_syscall_64`; return through SYSRETQ if eligible else IRET.
- [ ] AC-2: All entry paths from user perform swapgs BEFORE first GS-relative memory access.
- [ ] AC-3: All return-to-user paths perform SWITCH_TO_USER_CR3 BEFORE swapgs.
- [ ] AC-4: `paranoid_entry` for NMI/#MC/#DB/#DF reads MSR_GS_BASE; never blind-swapgs.
- [ ] AC-5: NMI nesting: nested NMI never overwrites outer NMI's pt_regs; outer NMI re-runs handler via repeat_nmi after nested completes.
- [ ] AC-6: INT 0x80 from 64-bit user delivers `do_int80_emulation` with 32-bit syscall table.
- [ ] AC-7: SYSENTER (32-bit compat) sets RFLAGS.IF before calling C dispatcher.
- [ ] AC-8: SYSRETQ rejected (fallthrough to IRET) if user RCX non-canonical.
- [ ] AC-9: KPTI: post-entry CR3 has kernel page tables; post-exit CR3 has user-mode page tables.
- [ ] AC-10: `entry_SYSCALL_64_after_hwframe` is a global symbol resolvable by /proc/kallsyms.
- [ ] AC-11: `error_entry` detects user-GS state when interrupted in the SYSCALL gap between MSR_LSTAR target and stack-switch.
- [ ] AC-12: `ret_from_fork_asm`: a newly forked userspace task returns to user_rip with all GPRs from `copy_thread`-prepared pt_regs.
- [ ] AC-13: IST entries (NMI/MCE/#DB/#DF): each uses a dedicated per-CPU IST stack from `cpu_entry_area.exception_stacks`.

## Architecture

```
[user code]
    |
    | SYSCALL (HW: load RIP=MSR_LSTAR, RCX=user-RIP, R11=user-RFLAGS,
    |          CS/SS from MSR_STAR; do NOT switch RSP)
    v
entry_SYSCALL_64:
    swapgs                            # GS: user → kernel (cpu_entry_area)
    SAVE_AND_SWITCH_TO_KERNEL_CR3     # KPTI: user CR3 → kernel CR3
    mov   %rsp, PER_CPU(cpu_tss_rw.x86_tss.sp2)   # stash user RSP
    mov   PER_CPU(pcpu_hot.top_of_stack), %rsp    # load kernel RSP
entry_SYSCALL_64_safe_stack:
    push  $__USER_DS                  # build IRET-compatible frame
    push  PER_CPU(.sp2)               # user RSP
    push  %r11                        # user RFLAGS
    push  $__USER_CS
    push  %rcx                        # user RIP
    push  $-1                         # orig_ax = -1 (sentinel)
    PUSH_AND_CLEAR_REGS rax=$-ENOSYS  # full pt_regs, RAX pre-set
entry_SYSCALL_64_after_hwframe:
    mov   %rsp, %rdi                  # arg1 = pt_regs
    mov   %eax, %esi                  # arg2 = nr
    call  do_syscall_64               # C dispatch
    # returns: AL=1 if SYSRETQ-eligible, AL=0 if must IRET

    test  %al, %al
    jz    swapgs_restore_regs_and_return_to_usermode
    # SYSRETQ fast path:
    POP_REGS pop_rdi=0
    SWITCH_TO_USER_CR3_STACK scratch_reg=%rdi
    pop  %rdi
    pop  %rsp                         # back to user RSP
entry_SYSRETQ_unsafe_stack:
    swapgs
    sysretq

# IRET slow path for incompatible RIP/RCX/RFLAGS:
swapgs_restore_regs_and_return_to_usermode:
    POP_REGS
    SWITCH_TO_USER_CR3 scratch_reg=%rdi scratch_reg2=%rax
    swapgs
native_irq_return_iret:
    iretq
```

```
NMI nesting state machine (asm_exc_nmi):

[NMI from user]                      [NMI from kernel]
        |                                    |
        v                                    v
  swapgs                            test_if_in_repeat_nmi_window
  SWITCH_TO_KERNEL_CR3                 |
  call exc_nmi                         | ↳ if in [repeat_nmi, end_repeat_nmi):
  jmp swapgs_restore_regs_…            |     nested! → modify outer iret to repeat_nmi
                                       |     → just iretq (nested returns)
                                       v
                                  first_nmi:
                                       copy HW NMI frame → per-CPU nmi_stack
                                       set "outermost" sentinel
                                       call exc_nmi
                                       restore CR3, swapgs-if-needed
                                       iretq    # ← may take a pending NMI here
                                  repeat_nmi:
                                       # re-runs handler with copied frame
                                       call exc_nmi
                                       iretq
                                  end_repeat_nmi:
```

```
paranoid_entry  (NMI, #MC, #DB, #DF):
    push %r12 / %r13 / %r14 (save scratch)
    SAVE_AND_SWITCH_TO_KERNEL_CR3 scratch_reg=%rax save_reg=%r14
    rdmsr MSR_GS_BASE
    test  high_bit                    # already kernel-GS?
    jnz   .Lskip_swapgs
    swapgs
.Lskip_swapgs:
    ret

paranoid_exit:                        # strict inverse of paranoid_entry
    restore CR3 from %r14
    test  saved_gs_state
    jz    .Lno_swapgs
    swapgs
.Lno_swapgs:
    pop  %r14 / %r13 / %r12
    ret
```

```
entry_64_compat.S layout:

entry_SYSENTER_compat:                # Intel 32-bit SYSENTER → MSR_IA32_SYSENTER_EIP
    swapgs
    SWITCH_TO_KERNEL_CR3
    # build 32-bit pt_regs; fix RFLAGS.IF
.Lsysenter_fix_flags:
    push  $X86_EFLAGS_FIXED
    popf
    jmp   .Lsysenter_flags_fixed
    → call do_fast_syscall_32

entry_SYSCALL_compat:                 # AMD 32-bit SYSCALL → MSR_CSTAR
    swapgs
    SWITCH_TO_KERNEL_CR3
    # build 32-bit pt_regs
    → call do_fast_syscall_32

entry_SYSRETL_compat_unsafe_stack:
    SWITCH_TO_USER_CR3_NOSTACK
    swapgs
    sysretl

int80_emulation:                      # legacy INT 0x80 (IDT vector 0x80, DPL=3)
    jmp   do_int80_emulation          # C truncates RAX → 32-bit syscall index
```

`arch::x86::entry::syscall_64` (entry_SYSCALL_64):
1. /* HW pre-state: GS=user, CR3=user, RSP=user, RCX=user-RIP, R11=user-RFLAGS */
2. swapgs — `wrgsbase` is not used; `swapgs` is atomic w.r.t. NMI delivery.
3. SAVE_AND_SWITCH_TO_KERNEL_CR3 — uses `pcpu_hot.user_pcid_flush_mask` (per-CPU, GS-relative).
4. Stash user RSP into `PER_CPU(cpu_tss_rw.x86_tss.sp2)`.
5. Load kernel RSP from `PER_CPU(pcpu_hot.top_of_stack)`.
6. Push synthetic IRET frame: SS=__USER_DS, RSP=user-RSP, RFLAGS=R11, CS=__USER_CS, RIP=RCX, orig_ax=-1.
7. PUSH_AND_CLEAR_REGS rax=-ENOSYS — pushes 15 GPRs; RAX pre-cleared to -ENOSYS so an out-of-range syscall returns -ENOSYS without further work.
8. Mark `entry_SYSCALL_64_after_hwframe` (global symbol for tooling).
9. Call `do_syscall_64(regs=%rsp, nr=%eax)`.
10. On return: test AL.
11. /* AL == 0 ⟹ IRET-only path (signal pending, non-canonical RCX, debug state, …) */
12. jz `swapgs_restore_regs_and_return_to_usermode`.
13. /* AL == 1 ⟹ SYSRETQ fast path */
14. POP_REGS pop_rdi=0.
15. SWITCH_TO_USER_CR3_STACK scratch_reg=%rdi.
16. pop %rdi.
17. pop %rsp — back to user RSP.
18. swapgs.
19. sysretq.

`arch::x86::entry::common_interrupt_return`:
1. /* Determine target: kernel or user, from saved CS */
2. testb $3, CS(%rsp).
3. jz `restore_regs_and_return_to_kernel`.
4. /* User return */
5. POP_REGS.
6. /* If X86_FEATURE_PTI: SWITCH_TO_USER_CR3 */
7. SWITCH_TO_USER_CR3 scratch_reg=%rdi scratch_reg2=%rax.
8. swapgs.
9. `native_irq_return_iret`: iretq. — exception-table-fixupable for #GP on IRET (e.g. bad SS/RSP); fixup re-takes #GP via `general_protection` after switching back.

`arch::x86::entry::paranoid_entry`:
1. /* Called by NMI / #MC / #DB / #DF; arrives with unknown GS state */
2. cld.
3. PUSH_AND_CLEAR_REGS save_ret=1.
4. SAVE_AND_SWITCH_TO_KERNEL_CR3 scratch_reg=%rax save_reg=%r14 — save prior PCID/CR3 state into R14.
5. mov $MSR_GS_BASE, %ecx.
6. rdmsr. — EAX:EDX = current GS base.
7. testl %edx, %edx — sign bit of high 32 bits: 1 ⟹ kernel-canonical (i.e. already kernel-GS).
8. js `.Lparanoid_kernel_gs`.
9. swapgs. — was user-GS; flip to kernel.
10. .Lparanoid_kernel_gs: ret.

`arch::x86::entry::paranoid_exit`:
1. /* Restore CR3 from R14 — uses save_reg semantics from paranoid_entry */
2. RESTORE_CR3 scratch_reg=%rax save_reg=%r14.
3. /* Decide swapgs */
4. testl $1, was_user_gs(%rsp) — sentinel from paranoid_entry (encoded in saved EBX or similar slot).
5. jz `.Lparanoid_no_swapgs`.
6. swapgs.
7. .Lparanoid_no_swapgs: jmp restore_regs_and_return_to_kernel.

`arch::x86::entry::asm_exc_nmi`:
1. /* HW: NMI delivered; HW masks NMI until next IRET; SS:RSP, RFLAGS, CS:RIP pushed */
2. /* If from kernel: check nesting */
3. cmpw $__KERNEL_CS, CS(%rsp).
4. jne `.Lnmi_from_user`.
5. /* Kernel NMI: are we already in an NMI handler? */
6. movq %rsp, %rdx.
7. movq $repeat_nmi, %rax. cmpq %rax, %rdx. ja `.Lnmi_outermost`.
8. movq $end_repeat_nmi, %rax. cmpq %rax, %rdx. jb `.Lnmi_outermost`.
9. /* RIP is in [repeat_nmi, end_repeat_nmi): we are inside another NMI handler.
    Treat as nested: rewrite outer NMI's saved RIP to repeat_nmi and just iretq. */
10. `nested_nmi`: modify outer iret frame: pushq $repeat_nmi as new RIP, then iretq.
11. `.Lnmi_outermost` / `first_nmi`: copy HW NMI frame to per-CPU nmi_stack; set up "outermost" sentinel; call exc_nmi; iretq (may take pending NMI here).
12. `repeat_nmi`: re-runs handler against the saved-frame copy.
13. `end_repeat_nmi`.
14. `.Lnmi_from_user`: swapgs; SWITCH_TO_KERNEL_CR3; PUSH_AND_CLEAR_REGS; mov %rsp, %rdi; call exc_nmi; jmp swapgs_restore_regs_and_return_to_usermode.

`arch::x86::entry::int80_emulation`:
1. /* HW: IDT vector 0x80, DPL=3 (user-invokable); arrived via interrupt gate, IF cleared */
2. ASM_CLAC. — clear AC flag (SMAP discipline).
3. jmp do_int80_emulation. — C-side handles 32-bit syscall convention.

`arch::x86::syscall::do_int80_emulation` (C):
1. /* Force 32-bit syscall ABI even from 64-bit user */
2. nr = (u32)regs.orig_ax. — truncate RAX → EAX.
3. instrumentation_begin().
4. if !syscall_enter_from_user_mode(regs, nr): return.
5. nr = array_index_nospec(nr, IA32_NR_syscalls).
6. regs.ax = ia32_sys_call_table[nr](regs).
7. instrumentation_end().
8. syscall_exit_to_user_mode(regs).

`arch::x86::entry::sysenter_compat`:
1. swapgs.
2. SWITCH_TO_KERNEL_CR3 scratch_reg=%rax.
3. /* HW gave us SS/SP from MSR_IA32_SYSENTER_ESP, CS/EIP from MSR_IA32_SYSENTER_EIP */
4. movq PER_CPU(cpu_tss_rw.x86_tss.sp1), %rsp. — switch to kernel SYSENTER stack.
5. push synthetic IRET frame for 32-bit compat: SS=__USER32_DS, RSP, RFLAGS, CS=__USER32_CS, EIP.
6. PUSH_AND_CLEAR_REGS rax=$-ENOSYS.
7. /* SYSENTER clears IF in HW; we need to restore it before C entry */
8. testl $X86_EFLAGS_IF, EFLAGS(%rsp).
9. jnz `.Lsysenter_flags_fixed`.
10. `.Lsysenter_fix_flags`: push $X86_EFLAGS_FIXED; popfq; jmp .Lsysenter_flags_fixed.
11. `.Lsysenter_flags_fixed`: movq %rsp, %rdi; call do_fast_syscall_32.

`arch::x86::syscall::do_fast_syscall_32` (C):
1. /* 32-bit fast syscall (SYSENTER or SYSCALL_compat) C dispatch */
2. nr = (u32)regs.orig_ax.
3. if !__do_fast_syscall_32(regs): /* fast-path fail */
4.   return false. — caller falls through to slow IRET return.
5. return true. — caller uses sysexit/sysretl fast return.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `swapgs_before_first_pcpu_access` | INVARIANT | per-entry-from-user: swapgs precedes any GS-relative read. |
| `cr3_swap_before_first_kernel_data` | INVARIANT | per-entry: SWITCH_TO_KERNEL_CR3 precedes any kernel-virtual deref. |
| `user_cr3_swap_before_return_swapgs` | INVARIANT | per-exit-to-user: SWITCH_TO_USER_CR3 precedes swapgs. |
| `pt_regs_complete_at_after_hwframe` | INVARIANT | at entry_SYSCALL_64_after_hwframe: all 15 GPRs + IRET frame valid in `struct pt_regs`. |
| `sysretq_eligibility_strict` | INVARIANT | per-do_syscall_64-AL==1: RIP canonical ∧ RIP==RCX ∧ R11==RFLAGS ∧ no TIF_NEED_RESCHED/SIGPENDING. |
| `paranoid_entry_gs_check_via_rdmsr` | INVARIANT | per-paranoid: GS detection via rdmsr MSR_GS_BASE, never via cached state. |
| `nmi_no_loss_under_nesting` | INVARIANT | per-NMI nesting: nested NMI causes outer handler to re-run via repeat_nmi; no NMI dropped. |
| `ist_stack_per_vector` | INVARIANT | per-IST vector (NMI/MCE/#DB/#DF): RSP after entry points into the per-CPU `exception_stacks` slot for that vector. |
| `int80_forces_32bit_table` | INVARIANT | per-INT-0x80: dispatch via ia32_sys_call_table, never x64. |
| `iret_fault_fixup_present` | INVARIANT | per-native_irq_return_iret: exception_table entry for #GP/#SS/#NP at iretq PC. |

### Layer 2: TLA+

`arch/x86/entry-64.tla`:
- Per-CPU model: GS_state ∈ {user, kernel}, CR3_state ∈ {user, kernel}, NMI_state ∈ {idle, first, repeat, nested}.
- Per-transition: SYSCALL_entry, IRET_to_user, NMI_arrive, NMI_iret, paranoid_entry, paranoid_exit.
- Properties:
  - `safety_no_kernel_data_with_user_cr3` — kernel-virtual access ⟹ CR3_state = kernel.
  - `safety_no_user_data_with_kernel_cr3` — copy_from/to_user ⟹ stac()'d window only.
  - `safety_swapgs_pairing` — every entry-swapgs is matched by exactly one exit-swapgs.
  - `safety_nmi_handler_completes` — every NMI delivered eventually runs `exc_nmi` (possibly via repeat_nmi after a nested one).
  - `liveness_syscall_returns` — every entry_SYSCALL_64 eventually returns to user.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `entry_SYSCALL_64` post: `do_syscall_64` called with valid `*pt_regs` | `arch::x86::entry::syscall_64` |
| `do_syscall_64` post: returns bool indicating SYSRETQ eligibility | `arch::x86::syscall::do_syscall_64` |
| `paranoid_entry` post: GS=kernel ∧ CR3=kernel ∧ R14=saved-CR3-state | `arch::x86::entry::paranoid_entry` |
| `paranoid_exit` post: GS,CR3 restored to pre-`paranoid_entry` state | `arch::x86::entry::paranoid_exit` |
| `error_entry` post: GS=kernel ∧ CR3=kernel regardless of entry source | `arch::x86::entry::error_entry` |
| `int80_emulation` post: `do_int80_emulation` called with 32-bit ABI semantics | `arch::x86::entry::int80_emulation` |
| `__switch_to_asm` post: RSP, RBP, RBX, R12-R15 restored from `next` task | `arch::x86::entry::switch_to_asm` |

### Layer 4: Verus/Creusot functional

`Per-syscall-entry: SYSCALL(MSR_LSTAR) → swapgs → SWITCH_TO_KERNEL_CR3 → stack-switch → pt_regs build → do_syscall_64 → SYSRETQ-or-IRET-to-user` semantic equivalence: per-Intel SDM Vol 2 (SYSCALL/SYSRET), per-Documentation/x86/entry_64.rst, per-Documentation/x86/kernel-stacks.rst.

`Per-NMI nesting: HW-NMI → asm_exc_nmi → (first_nmi | nested_nmi modify outer iret → repeat_nmi)` semantic equivalence: per-arch/x86/kernel/nmi.c comments + Documentation/x86/topology.rst NMI section.

## Hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

x86_64 entry reinforcement:

- **Per-swapgs-before-pcpu** — defense against per-Meltdown-class leak via wrong GS.
- **Per-PTI CR3 swap with PCID flush mask** — defense against per-Meltdown (CVE-2017-5754).
- **Per-paranoid-entry rdmsr GS check** — defense against per-NMI-in-swapgs-window (CVE-2014-9322 class).
- **Per-NMI nesting state machine** — defense against per-NMI loss under re-entry.
- **Per-IST stack per vector** — defense against per-stack-overflow cascade across #DB/#NMI/#DF/#MC.
- **Per-SAVE_AND_SWITCH_TO_KERNEL_CR3 with save_reg** — defense against per-nested-paranoid CR3 corruption.
- **Per-native_irq_return_iret exception fixup** — defense against per-malicious-IRET-frame #GP cascade.
- **Per-SYSRETQ canonical-RCX rejection (fallthrough to IRET)** — defense against per-CVE-2014-9322 sysret#GP escalation.
- **Per-PUSH_AND_CLEAR_REGS rax=-ENOSYS** — defense against per-uninitialized-reg leak across boundary.
- **Per-ASM_CLAC at INT-0x80 entry** — defense against per-SMAP-bypass via stale AC.
- **Per-cpu_entry_area read-only mapping in user CR3** — defense against per-user-page-tables-leak.
- **Per-SWITCH_TO_USER_CR3 strictly before swapgs on exit** — defense against per-leak-via-wrong-CR3 with kernel-GS.
- **Per-error_entry CS-gap detection** — defense against per-swapgs-skip when interrupted in SYSCALL gap.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization applied immediately after switching to the per-task kernel stack via `pcpu_hot.top_of_stack`; every entry point (SYSCALL_64, SYSENTER_compat, SYSCALL_compat, INT 0x80) gets a fresh offset.
- **PAX_MEMORY_STACKLEAK** — on exit-to-user (`swapgs_restore_regs_and_return_to_usermode`), unused kernel-stack frames are scrubbed to prevent stack-data leak across syscall.
- **PAX_KERNEXEC** — entry assembly is `.text` RX/RO; PUSH_AND_CLEAR_REGS pre-clears RAX = -ENOSYS so unknown syscall returns without leaking register state.
- **PAX_USERCOPY** — `pt_regs` from userspace is treated as untrusted; all user-space register reads are bounded.
- **PAX_UDEREF (SMAP/SMEP)** — `ASM_CLAC` issued at INT 0x80 and SYSCALL entry to clear stale AC flag; defense against SMAP-bypass.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `__switch_to_asm` → `__switch_to` C dispatch and on syscall-table indirect calls.
- **KPTI CR3 swap with PCID flush mask** — `SAVE_AND_SWITCH_TO_KERNEL_CR3` on every entry, `SWITCH_TO_USER_CR3` strictly-before swapgs on every exit; Meltdown defense (CVE-2017-5754).
- **paranoid_entry rdmsr GS check** — defense against NMI-in-swapgs-window (CVE-2014-9322 class); GS detected via `rdmsr MSR_GS_BASE`, never via blind swapgs.
- **NMI nesting state machine** — first_nmi / repeat_nmi / end_repeat_nmi sentinel; defense against NMI loss under re-entry.
- **IST stack per vector** — distinct per-CPU IST stacks for #DB / #NMI / #DF / #MC / #VC; defense against stack-overflow cascade.
- **GRKERNSEC_HIDESYM** — kernel-pointer hiding for `entry_SYSCALL_64_after_hwframe` etc. in /proc.
- **GRKERNSEC_DMESG** — restrict syslog output to CAP_SYSLOG.
- **SYSRETQ canonical-RCX rejection** — fallthrough to IRET on non-canonical user RCX (CVE-2014-9322 escalation defense).
- **PAX_REFCOUNT** — saturating refcount on per-CPU `cpu_entry_area` mapping references.
- **cpu_entry_area read-only mapping in user CR3** — defense against user-page-tables-leak of kernel internals.

Per-doc rationale: entry_64 is the single point at which user-mode and kernel-mode states meet on every syscall, IRQ, exception, and NMI; grsec/PaX hardening here is non-negotiable and overlapping (RANDKSTACK + STACKLEAK + KPTI + paranoid_entry + IST + RAP + KERNEXEC) because any failure in any of them is a complete privilege boundary collapse.

## Open Questions

- (none at this Tier-3 level; FRED-path entry covered by `entry_fred.c` and a future sibling Tier-3 if FRED upstream lands)

## Out of Scope

- FRED entry (`entry_64_fred.S` + `entry_fred.c`) — covered separately when promoted.
- IDT vector table population (`arch/x86/kernel/idt.c`) — covered by `arch/x86/traps.md`.
- Per-vector exception C handlers (`arch/x86/kernel/traps.c`) — covered by `arch/x86/traps.md`.
- Syscall table generation (`arch/x86/entry/syscalls/syscall_*.tbl`) — covered by `arch/x86/entry.md` Tier-2.
- Signal-frame construction (`arch/x86/kernel/signal_64.c` / `signal_32.c`) — covered by `arch/x86/signal.md`.
- vDSO / vvar (`arch/x86/entry/vdso/`) — covered by `arch/x86/vdso.md`.
- VMX / SVM guest entry (`arch/x86/kvm/`) — covered by KVM Tier-2.
- Implementation code.
