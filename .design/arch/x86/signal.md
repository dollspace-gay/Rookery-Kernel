# Tier-3: arch/x86/signal — signal frame, sigreturn, signal delivery

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - arch/x86/kernel/signal.c
  - arch/x86/kernel/signal_64.c
  - arch/x86/kernel/signal_32.c
  - arch/x86/kernel/fpu/signal.c
  - arch/x86/include/uapi/asm/sigcontext.h
  - arch/x86/include/uapi/asm/ucontext.h
  - arch/x86/include/uapi/asm/signal.h
  - arch/x86/include/asm/sighandling.h
  - arch/x86/entry/vdso/
-->

## Summary
Tier-3 design for x86_64 signal frame construction + sigreturn + per-signal delivery. Owns the userspace-visible `struct sigcontext`, `struct ucontext_t`, FPU/SIMD state save area, signal-handler call-frame setup, and the `rt_sigreturn` syscall handler (which restores user state from the signal frame). The signal restorer (the user-mode trampoline that the kernel jumps to before the user's signal handler returns) lives in the vDSO; cross-ref `arch/x86/vdso.md`.

This is the most user-visible contract for signal handling on x86_64. Every userspace process receives signals via this path; the byte-for-byte signal-frame layout MUST match upstream so existing signal handlers + ucontext-based coroutine libraries (libucontext, makecontext-based) work unmodified.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| 64-bit signal delivery + sigreturn | `arch/x86/kernel/signal.c`, `arch/x86/kernel/signal_64.c` |
| 32-bit (compat) signal delivery | `arch/x86/kernel/signal_32.c` |
| FPU save/restore for signal frames | `arch/x86/kernel/fpu/signal.c` |
| UAPI sigcontext + ucontext | `arch/x86/include/uapi/asm/sigcontext.h`, `arch/x86/include/uapi/asm/ucontext.h`, `arch/x86/include/uapi/asm/signal.h` |
| Internal helpers | `arch/x86/include/asm/sighandling.h` |
| vDSO signal restorer | `arch/x86/entry/vdso/` (cross-ref `arch/x86/vdso.md`) |

## Compatibility contract

### `struct sigcontext` (64-bit)

`arch/x86/include/uapi/asm/sigcontext.h` defines `struct sigcontext`. **Byte-identical** layout required:

| Field | Size | Offset (64-bit) |
|---|---|---|
| `r8`, `r9`, `r10`, `r11`, `r12`, `r13`, `r14`, `r15` | 8 × 8 bytes | 0..63 |
| `rdi`, `rsi`, `rbp`, `rbx`, `rdx`, `rax`, `rcx`, `rsp`, `rip`, `eflags` | 10 × 8 bytes | 64..143 |
| `cs`, `gs`, `fs`, `__pad0` | 4 × 2 bytes | 144..151 |
| `err`, `trapno`, `oldmask`, `cr2` | 4 × 8 bytes | 152..183 |
| `fpstate` pointer | 8 bytes | 184..191 |
| Reserved | 64 bytes (8 × 8-byte words) | 192..255 |

Total: 256 bytes. (Note: x32 ABI variant — out of scope per `arch/x86/00-overview.md` D1.)

### `struct ucontext_t`

`arch/x86/include/uapi/asm/ucontext.h` defines `struct ucontext_ia32` (32-bit) and the 64-bit variant. The 64-bit version:

```c
struct ucontext_t {
    unsigned long uc_flags;
    struct ucontext_t *uc_link;
    stack_t uc_stack;
    struct sigcontext uc_mcontext;
    sigset_t uc_sigmask;
};
```

Byte-identical layout.

### FPU state in signal frame

The `sigcontext.fpstate` pointer references a per-task FPU state buffer (XSAVE-formatted) saved on the user stack. Layout matches upstream's `xstate_layout`; size depends on enabled XSAVE features (CPUID 0xD lookup).

### `siginfo_t`

`include/uapi/asm-generic/siginfo.h` (cross-arch) defines `siginfo_t`. The first 3 fields (`si_signo`, `si_errno`, `si_code`) are universally present. The union of remaining fields varies by signal type. Byte-identical layout required.

### Signal restorer

The vDSO exports `__kernel_rt_sigreturn` — the user-mode trampoline that the kernel sets as the return-from-signal-handler RIP. Its contents:

```asm
__kernel_rt_sigreturn:
    movq    $__NR_rt_sigreturn, %rax
    syscall
```

(or `__kernel_sigreturn` for the legacy 32-bit signal frame.)

The vDSO + the signal-frame layout together define the complete signal-handling ABI per upstream's stable-ABI guarantee.

### Signal-stack setup

`SA_ONSTACK` flag + `sigaltstack(2)` syscall switch the signal handler to an alternate stack. Per-task `signal_stack` field; semantics byte-identical to upstream.

## Requirements

- REQ-1: `struct sigcontext` 64-bit layout byte-identical (per the table above).
- REQ-2: `struct sigcontext_32` (compat) layout byte-identical.
- REQ-3: `struct ucontext_t` layout byte-identical.
- REQ-4: FPU state in signal frame: XSAVE-format buffer; size + layout per CPUID 0xD; sub-features per upstream's `XSAVE_HEADER_OFFSET` table.
- REQ-5: Signal-frame construction on user stack: per upstream's `setup_rt_frame` algorithm. `pretcode` set to vDSO `__kernel_rt_sigreturn`.
- REQ-6: `rt_sigreturn` syscall: pop the frame off user stack, validate `siginfo` + `ucontext_t` layouts, restore user pt_regs from the frame, restore FPU state, deliver next pending signal if any.
- REQ-7: `siginfo_t` layout byte-identical for every signal-specific union variant (SIGSEGV's `si_addr` + `si_addr_lsb` + `si_pkey`; SIGCHLD's `si_status` + `si_utime` + `si_stime`; etc.).
- REQ-8: SA_ONSTACK + sigaltstack semantics byte-identical: alternate-stack switch on signal delivery; sigaltstack(2) updates per-task `signal_stack`.
- REQ-9: Signal blocking: `signal_struct->blocked` mask honored; SA_NODEFER honored; SA_RESETHAND honored.
- REQ-10: Synchronous signal delivery from CPU exceptions (SIGSEGV/SIGBUS/SIGFPE/SIGILL): `sigcontext.err`, `sigcontext.trapno`, `sigcontext.cr2` populated per upstream — matters for handlers that introspect.
- REQ-11: x86 syscall-restart logic: `SA_RESTART` flag + `do_signal_pending` checks; only restartable syscalls restart per upstream's `signal_pt_regs` algorithm.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct sigcontext` produces byte-identical layout vs. upstream. (covers REQ-1, REQ-2)
- [ ] AC-2: `pahole struct ucontext_t` produces byte-identical layout. (covers REQ-3)
- [ ] AC-3: A test installs a signal handler, triggers a signal, prints `mcontext->gregs[0..16]` field-by-field; output byte-identical vs. upstream. (covers REQ-1, REQ-3)
- [ ] AC-4: An AVX-512-using process receives a signal and its handler reads `mcontext->fpregs->_xmm[0..15]` + `_st[0..7]` + the high-component XSAVE area; data round-trip preserves identical bytes. (covers REQ-4)
- [ ] AC-5: A test that sigsetjmp + siglongjmp via the libc-defined `setcontext`/`getcontext` round-trips state correctly under Rookery (matches upstream). (covers REQ-5, REQ-6)
- [ ] AC-6: Signal selftests under `tools/testing/selftests/signals/` pass with the same set as upstream. (covers REQ-7, REQ-8, REQ-9)
- [ ] AC-7: A SIGSEGV from a NULL deref produces a siginfo_t with `si_addr = 0x0`, `si_signo = SIGSEGV`, `si_code = SEGV_MAPERR`, plus correct `sigcontext.{err, trapno, cr2}`. (covers REQ-10)
- [ ] AC-8: A read(2) on a slow file blocked when signal arrives + handler runs: with SA_RESTART, the syscall restarts; without, it returns EINTR. (covers REQ-11)
- [ ] AC-9: A `swapcontext` (libucontext) test program switches between coroutines correctly under Rookery. (covers REQ-3, REQ-5)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::arch::x86::signal::frame` — sigcontext + ucontext frame builders
- `kernel::arch::x86::signal::deliver` — signal-handler call-frame setup
- `kernel::arch::x86::signal::sigreturn` — `rt_sigreturn` syscall handler
- `kernel::arch::x86::signal::compat32` — 32-bit compat signal frames
- `kernel::arch::x86::signal::fpu_save` — FPU state save into the signal frame (cross-ref `arch/x86/kernel-platform.md` § fpu)

### Locking and concurrency

Signal delivery runs in the context of the receiving task; no cross-task locks. Per-task siglock (`task->sighand->siglock`) held while reading + clearing the pending-signal queue (cross-ref `kernel/task-lifecycle.md`).

The signal-frame is written to the user stack — userspace, not kernel memory. STAC/CLAC bracket the writes (cross-ref `lib/usercopy.md`).

### Error handling

Signal-frame setup errors are typically fatal (the receiving task gets SIGSEGV from `force_sigsegv`):
- Bad user stack (unmapped or write-protected) → SIGSEGV
- FPU buffer too small → SIGSEGV (matches upstream)
- Signal-stack overflow → SIGSEGV

`rt_sigreturn` validation errors are also fatal (forced SIGSEGV with detailed dmesg):
- `sigcontext.cs` not USER_CS → SIGSEGV
- `sigcontext.eflags` reserved bits set → SIGSEGV
- FPU state buffer corrupted → SIGSEGV

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Signal-frame writes to user stack (with STAC/CLAC bracketing) | `kani::proofs::arch::x86::signal::frame_write_safety` |
| sigreturn validation: cs, eflags, fpstate pointer | `kani::proofs::arch::x86::signal::sigreturn_validate_safety` |
| FPU XSAVE/XRSTOR with buffer-size checks | `kani::proofs::arch::x86::fpu::xstate_safety` (inherited from `arch/x86/entry.md`) |

### Layer 2: TLA+ models

(none mandatory — signal delivery is per-task; no novel concurrency)

### Layer 3: Kani harnesses for data-structure invariants

- `sigcontext` layout invariants — every field at the documented offset; checked at compile time via const assertions
- `ucontext_t` layout invariants — same approach

### Layer 4: Functional correctness (opt-in)

- **`rt_sigreturn` validation correctness** via Verus — proves: any sigcontext that passes validation can be safely restored without privilege escalation. High-leverage; sigreturn is historically a major attack surface (sigreturn-oriented programming).

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CET shadow stack** | Per-task shadow stack switched when signal arrives; restored on sigreturn | (cross-ref `arch/x86/cpu-mitigations.md`) |

### Row-1 features consumed by this component

- **UDEREF**: signal-frame writes go through `UserPtr<Sigcontext>` typed accessor (no raw deref)
- **CLOSE_KERNEL/CLOSE_USERLAND**: STAC/CLAC bracket all signal-frame user-stack writes (consumed from `arch/x86/cpu-mitigations.md`)
- **SIZE_OVERFLOW**: signal-stack offset arithmetic uses checked operators
- **CONSTIFY**: signal-default-action table is `static const`

### Row-2 / GR-RBAC integration

Signal delivery is post-`security_task_kill` LSM hook (handled in `kernel/task-lifecycle.md`); this component does not invoke LSM hooks itself.

### Userspace-visible behavior changes

None beyond upstream defaults. Signal-frame layout is byte-identical.

### Verification

(See § Verification above.)

## Open Questions

(none — signal-frame layout is contractually rigid via the userspace ABI guarantee)

## Out of Scope

- Cross-arch signal infrastructure (cross-ref `kernel/task-lifecycle.md` § signal)
- 32-bit-only signal paths
- Implementation code
