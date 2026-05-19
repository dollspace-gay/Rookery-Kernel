---
title: "Tier-5 syscall: rt_sigreturn(2) — syscall 15"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`rt_sigreturn(2)` is the **signal-trampoline return** syscall: it tears down the kernel-built signal frame on the user stack, restores all general-purpose registers, the floating-point/vector state, the signal mask, the alt-stack pointer, and resumes execution at the pre-signal IP. Userspace never calls `rt_sigreturn` directly — the kernel installs a trampoline (either in the user's signal handler context via `sa_restorer`, or in the VDSO `__kernel_rt_sigreturn`) and arranges for the handler's `ret` to land on the trampoline, which immediately executes `syscall` with `__NR_rt_sigreturn` in `%rax`.

Because every byte of the signal frame is attacker-readable (the user can mprotect their stack), the kernel must rebuild kernel-side registers from a fully-attacker-controlled `struct ucontext` while refusing any combination that would elevate privilege, escape segmentation, or unwind a non-signal context. This is one of the historically most exploited syscalls (sigreturn-oriented-programming, SROP).

Critical for: every signal handler in every program on Linux (since 2.2; rt_ extends the original sigreturn for POSIX rt-signals 32-64 and siginfo_t).

### Acceptance Criteria

- [ ] AC-1: Signal handler returns normally: rt_sigreturn restores all GPRs and resumes at pre-signal IP.
- [ ] AC-2: Signal handler modifies ucontext.uc_mcontext.rip: thread resumes at modified IP.
- [ ] AC-3: Signal handler corrupts ucontext.uc_mcontext.cs to __KERNEL_CS: SIGSEGV.
- [ ] AC-4: Signal handler corrupts ucontext.uc_mcontext.rip to non-canonical address: SIGSEGV.
- [ ] AC-5: Signal handler corrupts ucontext.uc_mcontext.fpstate.xstate_bv reserved bit: SIGSEGV.
- [ ] AC-6: Signal mask restored to pre-signal mask after handler.
- [ ] AC-7: SIGKILL / SIGSTOP cannot be masked via rt_sigreturn.
- [ ] AC-8: rt_sigreturn from non-signal context with a hand-crafted frame: behaves as SROP gadget; SMAP / sigframe validation should detect when malformed.
- [ ] AC-9: TIF_SIGPENDING after rt_sigreturn delivers any queued signals correctly.
- [ ] AC-10: rt_sigreturn from alt-stack frame restores main stack pointer.
- [ ] AC-11: FPU state larger than xsave buffer: SIGSEGV.
- [ ] AC-12: rflags IOPL bits cannot be raised via rt_sigreturn.

### Architecture

```rust
#[syscall(nr = 15, abi = "sysv", noreturn)]
pub fn sys_rt_sigreturn() -> ! {
    Signal::do_rt_sigreturn(current_regs_mut())
}
```

`Signal::do_rt_sigreturn(regs) -> !`:
1. let sp = regs.sp;
2. let frame_ptr = UserPtr::<RtSigframe>::from(sp - SIGFRAME_OFFSET);
3. let frame: RtSigframe = match frame_ptr.copy_in_struct() {
4.   Ok(f) => f,
5.   Err(_) => Signal::force_sigsegv(),
6. };
7. /* Validate selectors */
8. if !Signal::is_user_cs(frame.uc.uc_mcontext.cs) { Signal::force_sigsegv(); }
9. if !Signal::is_user_ss(frame.uc.uc_mcontext.ss) { Signal::force_sigsegv(); }
10. /* Validate IP canonical (x86_64) */
11. if !Vmem::is_canonical(frame.uc.uc_mcontext.rip) { Signal::force_sigsegv(); }
12. /* Restore GPRs */
13. Signal::restore_sigcontext(&mut *regs, &frame.uc.uc_mcontext);
14. /* Restore FPU/XSAVE — validates xstate_header */
15. if let Err(_) = Signal::restore_fpstate(&frame.uc.uc_mcontext.fpstate) {
16.   Signal::force_sigsegv();
17. }
18. /* Restore signal mask, minus SIGKILL/SIGSTOP */
19. let mut mask = frame.uc.uc_sigmask;
20. mask.unblock(SIGKILL);
21. mask.unblock(SIGSTOP);
22. set_current_blocked(&mask);
23. /* Restore alt-stack */
24. Signal::restore_altstack(&frame.uc.uc_stack);
25. /* Normal syscall-exit path runs after we return-to-user */
26. /* This function does not return to its C caller; regs are now sigcontext-restored */
27. unreachable!("returns via syscall-exit assembly using restored regs");

`Signal::restore_sigcontext(regs, sc)`:
1. regs.r15 = sc.r15; regs.r14 = sc.r14; ... regs.rax = sc.rax;
2. regs.rip = sc.rip;
3. /* rflags: sanitize */
4. let mut flags = sc.rflags;
5. flags &= !(RFLAGS_IOPL_MASK | RFLAGS_VM | RFLAGS_NT);
6. flags |= regs.rflags & RFLAGS_IF;   // preserve kernel's IF
7. regs.rflags = flags;

`Signal::restore_fpstate(fpstate_ptr) -> Result<()>`:
1. let hdr: XStateHeader = fpstate_ptr.copy_in_struct()?;
2. if hdr.xfeatures & !host_xfeatures_supported() != 0 { return Err(()); }
3. if hdr.reserved.iter().any(|&w| w != 0)             { return Err(()); }
4. let size = hdr.xstate_size();
5. if size > MAX_XSAVE_BYTES                           { return Err(()); }
6. /* Copy into thread's xsave area via XRSTOR; SMAP-guarded */
7. fpu_state::restore_from_user(fpstate_ptr, size)?;
8. Ok(())

`Signal::force_sigsegv() -> !`:
1. force_sig_info(SIGSEGV, current());
2. /* schedule_exit / regenerated context will run; not a return */
3. unreachable!();

### Out of Scope

- Per-arch signal-frame setup (`__setup_rt_frame`) — Tier-3 per arch.
- `sigreturn(2)` legacy (covered separately).
- `sigaction(2)` (separate Tier-5 doc).
- FPU/XSAVE/AVX layout (Tier-3 `arch/x86/fpu.md`).
- Implementation code.

### signature

```c
/* Not exposed in userspace headers. The libc wrapper is `__restore_rt` (the
   trampoline) which executes the syscall. Userspace never types `rt_sigreturn`. */

int rt_sigreturn(void);

/* The frame on the user stack the kernel reads: */
struct rt_sigframe {
    char __user *pretcode;   /* unused on x86_64 — trampoline lives in VDSO */
    struct ucontext uc;
    struct siginfo info;
    /* FPU state may live within uc.uc_mcontext.fpstate, pointed via fpstate */
};

struct ucontext {
    unsigned long  uc_flags;
    struct ucontext *uc_link;
    stack_t        uc_stack;        /* alt-stack pointer/size/flags */
    struct sigcontext uc_mcontext;  /* per-arch register set */
    sigset_t       uc_sigmask;      /* mask to restore */
};
```

### parameters

(none — `SYSCALL_DEFINE0(rt_sigreturn)`. All state is read from the user stack pointed-to by `%rsp` at entry.)

### return value

`rt_sigreturn` does not return to its caller. On success, control resumes at the pre-signal IP with all registers restored. On failure, the kernel sends `SIGSEGV` (force_sig) to the calling thread and does not return either (handler context destroyed).

The `regs->ax` value written by the syscall is whatever the user supplied in `uc.uc_mcontext.rax` — i.e. the caller controls the post-resume `%rax`. This is sometimes used by signal handlers to override syscall return values when resuming a `SA_RESTART`-interrupted syscall.

### errors

| Effect | Trigger |
|---|---|
| `SIGSEGV` (force_sig_info) | Signal frame on user stack faults, or has invalid segment registers / IP outside user range / FPU state too large / unaligned `%rsp` / corrupted FPU magic. |
| Resume at user IP | Success; user resumes with restored register file. |

`rt_sigreturn` does NOT return `-1`/errno like a normal syscall. From userspace's perspective, the syscall instruction never returns (the kernel restores all GPRs including `%rax`).

### abi surface

```text
__NR_rt_sigreturn  (x86_64)   = 15
__NR_rt_sigreturn  (i386)     = 173
__NR_rt_sigreturn  (arm64)    = 139
__NR_rt_sigreturn  (riscv64)  = 139

/* On x86_64 the trampoline is in the VDSO:
   __kernel_rt_sigreturn:
       movq $__NR_rt_sigreturn, %rax
       syscall                          */

struct sigcontext {
    /* per-arch GPR snapshot; x86_64 is 23 fields including rip, rflags,
       cs, ss, fs_base, gs_base, fpstate, ... */
};
```

### compatibility contract

REQ-1: Syscall number is **15** on x86_64; 139 on arm64. ABI-stable since the rt-signal split (Linux 2.2).

REQ-2: rt_sigreturn is **only callable in signal-handler-return context**. The kernel does not refuse arbitrary callers — userspace may invoke `rt_sigreturn` from any context — but the syscall consumes the kernel-frame-relative `regs->sp` pointing at a struct rt_sigframe. If the user constructs a synthetic frame, the syscall behaves as a SROP gadget: this is intentional ABI but heavily hardened (SMAP, sigframe validation, SMEP at kernel-side).

REQ-3: The frame layout the kernel reads is per-arch:
  - x86_64: `regs->sp` MUST point at the byte just after the saved `pretcode` (signal-frame layout per `arch/x86/kernel/signal_64.c::__setup_rt_frame`).
  - arm64: `regs->sp` MUST point at `struct rt_sigframe { struct siginfo info; struct ucontext uc; }`.

REQ-4: Restored register file source:
  - GPRs: from `ucontext.uc_mcontext.{rax,rbx,rcx,rdx,rsi,rdi,rbp,rsp,r8..r15,rip,rflags,cs,ss,...}`.
  - Signal mask: from `ucontext.uc_sigmask` via `set_current_blocked(&mask)`.
  - Alt-stack: from `ucontext.uc_stack` via `do_sigaltstack()`.
  - FPU/XSAVE: from `ucontext.uc_mcontext.fpstate` (user-pointer; copied to kernel xsave region after validation).

REQ-5: Segment selectors `cs` and `ss` MUST be restored to user values; `__USER_CS` and `__USER_DS` enforced. Any kernel-level selector ⟹ SIGSEGV.

REQ-6: `rflags`: restored except sensitive bits — IF (interrupt-enable, bit 9), IOPL bits, VM, TF clear-on-restore (TF preserved if user previously had it set in trap context). NT (nested-task) is cleared.

REQ-7: `cs.rpl == 3` and `ss.rpl == 3` enforced (user privilege level).

REQ-8: IP (`rip`) MUST be in canonical-form for x86_64 (high bits sign-extended) — bad IP ⟹ #GP at iret, kernel catches → SIGSEGV.

REQ-9: FPU state validation: the `xstate_header` magic must match XSAVE area; `xfeatures` bits must be subset of host-supported; any reserved bit set ⟹ SIGSEGV. Defense against XSAVE-state-confusion exploits.

REQ-10: Signal mask: never includes SIGKILL or SIGSTOP (kernel masks those bits out).

REQ-11: Alt-stack restore: only if signal was delivered on alt-stack; otherwise current alt-stack preserved.

REQ-12: After restore, kernel proceeds through normal syscall-exit path: handles TIF_NEED_RESCHED, TIF_SIGPENDING (a new signal may be queued from the just-completed handler), TIF_NOTIFY_RESUME (rseq, io_uring task work).

REQ-13: x32 ABI and i386 compat: a separate `compat_rt_sigreturn` path exists; layout differs.

REQ-14: No syscall return value is "returned" to userspace via `%rax` directly — `%rax` is restored from the sigcontext. This is the only Linux syscall with this property (alongside `sigreturn` legacy).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `user_selectors_only` | INVARIANT | cs.rpl == 3 ∧ ss.rpl == 3 post-restore. |
| `ip_canonical` | INVARIANT | restored rip is canonical. |
| `rflags_sanitized` | INVARIANT | IOPL/VM/NT bits zero post-restore. |
| `mask_excludes_kill_stop` | INVARIANT | restored mask cannot block SIGKILL/SIGSTOP. |
| `fpstate_xfeatures_subset` | INVARIANT | xfeatures ⊆ host-supported. |
| `frame_copy_smap_guarded` | INVARIANT | all reads go through copy_from_user (SMAP). |
| `force_segv_on_invalid` | INVARIANT | any validation failure ⟹ force_sigsegv (not silent corruption). |

### Layer 2: TLA+

`arch/x86/rt_sigreturn.tla`:
- States: per-frame-fetch, per-validate, per-gpr-restore, per-fpu-restore, per-mask-restore, per-return-to-user.
- Properties:
  - `safety_no_kernel_selector` — restored cs/ss cannot be __KERNEL_*.
  - `safety_iopl_no_raise` — rflags.iopl post ≤ pre.
  - `safety_kill_stop_unmaskable` — mask never blocks SIGKILL/SIGSTOP.
  - `safety_fpstate_validated` — restore_fpstate fails ⟹ SIGSEGV.
  - `liveness_restore_or_die` — every entry terminates (either resumes user or delivers SIGSEGV).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_rt_sigreturn` post: regs.cs.rpl == 3 ∨ SIGSEGV delivered | `Signal::do_rt_sigreturn` |
| `restore_sigcontext` post: regs.rflags & FORBIDDEN_BITS == 0 | `Signal::restore_sigcontext` |
| `restore_fpstate` post: xstate area is sanitized subset of host | `Signal::restore_fpstate` |

### Layer 4: Verus / Creusot functional

Per-`sigreturn(2)` man-page; LTP `rt_sigreturn01`; selftests/sigaltstack pass; gdb signal-handler step-over works.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`rt_sigreturn(2)` reinforcement:

- **Per-frame SMAP-guarded copy** — defense against per-kernel-side read of unprotected user memory.
- **Per-selector RPL enforcement** — defense against per-cs/ss kernel-mode escalation.
- **Per-IP canonical-form check** — defense against per-non-canonical IP iret #GP exploit.
- **Per-rflags sanitization** — defense against per-IOPL raise.
- **Per-mask cannot block SIGKILL/SIGSTOP** — defense against per-undetachable-process.
- **Per-fpstate xfeatures subset** — defense against per-XSAVE-state-confusion (CVE class).
- **Per-fpstate reserved-bits zero** — defense against per-future-XSAVE-bit smuggling.

### grsecurity / pax surface

- **rt_sigreturn only callable via signal-frame return path** — grsec tags TIF_SIGFRAME_PENDING on signal delivery; rt_sigreturn invocation without that tag is treated as a SROP attempt and emits an audit record + SIGSEGV. Userspace constructing synthetic frames (legitimate libc use is via the trampoline only) is rejected.
- **PaX UDEREF on sigframe copy** — SMAP-enforced; sigframe copy is the most-stressed copy_from_user path in the kernel.
- **PaX KERNEXEC for VDSO trampoline** — the `__kernel_rt_sigreturn` page is RX-only; trampoline cannot be modified.
- **PAX_MEMORY_SANITIZE on signal-frame stack** — kernel-side sigframe build clears stack region pre-fill.
- **GRKERNSEC_KSTACKOVERFLOW** — sigreturn does not allocate kernel stack; bounded.
- **PaX RAP / CFI on restore_sigcontext** — restore path is short-lived; ROP cannot pivot through it.
- **GRKERNSEC_HIDESYM** — signal-frame layout symbols not exposed.
- **PAX_REFCOUNT neutral** — no refcount changes.
- **No_new_privs neutral** — rt_sigreturn is not a privilege boundary; the sanitization is the privilege boundary.
- **GRKERNSEC_AUDIT_SIGRETURN** — every SROP-suspect rt_sigreturn (no TIF_SIGFRAME_PENDING) is logged with the synthesized frame for forensic.
- **CAP requirement: none** — but a process flagged with TPE+brute audit emits richer rt_sigreturn logs.
- **xstate validation reinforced** — grsec rejects any frame whose xstate_bv is non-subset of host-supported, not merely the kernel default; closes the historic XSAVE state-confusion exploit class.

