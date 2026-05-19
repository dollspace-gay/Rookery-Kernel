# Tier-5 syscall: sigaltstack(2) — syscall 131

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/signal.c (sys_sigaltstack, do_sigaltstack)
  - include/linux/sched/signal.h (task_struct.sas_ss_*)
  - include/uapi/asm-generic/signal.h (stack_t, SS_ONSTACK, SS_DISABLE, SS_AUTODISARM)
  - arch/x86/entry/syscalls/syscall_64.tbl (131  common  sigaltstack)
-->

## Summary

`sigaltstack(2)` registers (or queries) a per-thread **alternate
signal stack** — a memory region the kernel switches to as the
hardware stack-pointer before invoking a signal handler installed
with `SA_ONSTACK`. The alternate stack is the only safe vehicle for
delivering `SIGSEGV`-on-stack-overflow handlers (the primary stack
is by definition unavailable then).

Per-thread state: `ss_sp` base address, `ss_size` (>= `MINSIGSTKSZ`),
`ss_flags`:
- `SS_DISABLE` (2): no alt stack registered.
- `SS_ONSTACK` (1): currently executing on the alt stack.
- `SS_AUTODISARM` (0x80000000): alt stack auto-disarms to `SS_DISABLE`
  for the duration of a handler that switches onto it; re-armed by
  sigreturn. Permits handlers to recursively `sigaltstack` or
  `makecontext` away without leaking state.

Setting a new stack while currently `SS_ONSTACK` is `-EPERM` unless
`SS_AUTODISARM` is used.

Critical for: every userspace runtime that handles SIGSEGV robustly
(Go, Rust panic-on-stack-overflow, JVM, V8); every fiber/coroutine
library needing known-good signal context; gdb/lldb in handler-trace
mode.

This Tier-5 covers `kernel/signal.c::SYSCALL_DEFINE2(sigaltstack, ...)`.

## Signature

```c
long sigaltstack(const stack_t *ss, stack_t *oss);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `ss` | `const stack_t *` | in | New alt-stack descriptor. `NULL` = query only. |
| `oss` | `stack_t *` | out | Previous alt-stack descriptor. `NULL` = do not return. |

## Return value

| Value | Meaning |
|---|---|
| `0`  | Success. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT`  | `ss` or `oss` non-NULL and not accessible. |
| `EINVAL`  | `ss.ss_flags` contains bits other than `SS_DISABLE | SS_AUTODISARM`. |
| `ENOMEM`  | `ss.ss_size < MINSIGSTKSZ` (and `SS_DISABLE` not set). |
| `EPERM`   | Currently on alt stack (`SS_ONSTACK`) and setting new stack without `SS_AUTODISARM`. |

## ABI surface

```text
__NR_sigaltstack (x86_64)    = 131
__NR_sigaltstack (i386)      = 186
__NR_sigaltstack (generic)   = 132   /* arm64/riscv/loongarch */
__NR_sigaltstack (powerpc)   = 185
__NR_sigaltstack (s390x)     = 186
__NR_sigaltstack (sparc)     = 76
```

### `stack_t`

```c
typedef struct sigaltstack {
    void  *ss_sp;        /* base address */
    int    ss_flags;     /* SS_DISABLE | SS_AUTODISARM */
    size_t ss_size;      /* size in bytes */
} stack_t;
```

LP64: 24 bytes (8 + 4 + 4 padding + 8). 32-bit: 12 bytes.

### Constants

```text
SS_ONSTACK     1            /* status: currently on alt stack */
SS_DISABLE     2            /* request: disable; or status: no alt stack */
SS_AUTODISARM  0x80000000   /* request: auto-disarm during handler */
MINSIGSTKSZ    2048 (x86_64) / 5120 (arm64) / 8192 (riscv64)
SIGSTKSZ       8192         /* recommended default */
```

## Compatibility contract

REQ-1: Syscall **131** on x86_64; **132** on generic. ABI-stable.

REQ-2: `stack_t` 24 bytes on LP64; 12 bytes on 32-bit. Compat handles conversion.

REQ-3: Both `ss` and `oss` NULL: return 0 immediately.

REQ-4: If `oss` non-NULL populate from current task:
- `oss.ss_sp = task.sas_ss_sp`, `oss.ss_size = task.sas_ss_size`.
- `oss.ss_flags`:
  - `SS_DISABLE` if `sas_ss_size == 0`.
  - Else `SS_ONSTACK` if `on_sig_stack(current_sp)`.
  - Else `0`.
  - OR `SS_AUTODISARM` if set.

REQ-5: If `ss` non-NULL: copy from user, validate, install.

REQ-6: Currently-on-stack guard: on alt stack AND `ss != NULL` AND new `ss.ss_flags & SS_AUTODISARM == 0` → `-EPERM`. Prevents pulling the rug out from in-handler execution.

REQ-7: `ss_flags` validation: only `SS_DISABLE | SS_AUTODISARM` permitted; any other bit → `-EINVAL`.

REQ-8: `SS_DISABLE`: clear `sas_ss_sp/size/flags`; return 0.

REQ-9: Without `SS_DISABLE`:
- `ss.ss_size >= MINSIGSTKSZ_arch` else `-ENOMEM`.
- Install `sas_ss_sp = ss.ss_sp`, `sas_ss_size = ss.ss_size`, `sas_ss_flags = ss.ss_flags & SS_AUTODISARM` (other bits stripped).
- Return 0.

REQ-10: Per-thread, not per-process. `fork()` copies state; `clone(CLONE_VM)` does NOT share (each thread registers independently). `execve()` resets to `SS_DISABLE`.

REQ-11: Sigframe-setup interaction (Tier-3 detail): on signal delivery whose `sa_flags & SA_ONSTACK` is set AND alt stack registered AND not already on alt stack: sigframe placed at `(sas_ss_sp + sas_ss_size - sigframe_size) & ~(stack_align - 1)`. If `SS_AUTODISARM`, save `sas_ss_*` into sigframe and clear; sigreturn restores them.

REQ-12: `on_sig_stack(sp)` iff `sas_ss_sp <= sp < sas_ss_sp + sas_ss_size`. `sp` is userspace `RSP` at syscall entry (pt_regs).

REQ-13: Audit: `AUDIT_SYSCALL` captures requested `ss_sp/size/flags` and prior state via `oss`.

REQ-14: `MINSIGSTKSZ` enforced strictly — no rounding up.

REQ-15: `ss_sp` is NOT validated against address space at install time (intentional — may be lazily mapped). Validation happens at sigframe setup; inaccessible alt stack causes recursive SIGSEGV that terminates the process.

REQ-16: Compat 32-bit: `compat_sigaltstack` narrows `ss_sp` to 32-bit pointer.

## Acceptance Criteria

- [ ] AC-1: Syscall number 131 on x86_64; 132 on generic.
- [ ] AC-2: Register SIGSTKSZ alt stack; install SIGSEGV handler with SA_ONSTACK; trigger stack overflow; handler runs on alt stack (`on_sig_stack() == true`).
- [ ] AC-3: ss_size = MINSIGSTKSZ - 1: `-ENOMEM`.
- [ ] AC-4: ss_flags has unknown bit 0x4: `-EINVAL`.
- [ ] AC-5: Query-only (ss=NULL, oss non-NULL): returns prior state.
- [ ] AC-6: Both NULL: returns 0; no state change.
- [ ] AC-7: In handler on alt stack, sigaltstack({newss, 0, ...}) without SS_AUTODISARM: `-EPERM`.
- [ ] AC-8: In handler, sigaltstack({newss, SS_AUTODISARM, ...}): success.
- [ ] AC-9: SS_AUTODISARM: handler entry clears `sas_ss_*`; sigreturn restores.
- [ ] AC-10: SS_DISABLE: clears state; oss returns SS_DISABLE.
- [ ] AC-11: fork() copies alt-stack state; clone(CLONE_VM) does not share.
- [ ] AC-12: execve() resets alt stack to SS_DISABLE.
- [ ] AC-13: `ss` faulting page: `-EFAULT`; no state change.
- [ ] AC-14: Multiple thread registrations independent.

## Architecture

```rust
#[syscall(nr = 131, abi = "sysv")]
pub fn sys_sigaltstack(
    ss:  UserPtr<UapiStack>, oss: UserPtr<UapiStack>,
) -> isize {
    Signal::do_sigaltstack(ss, oss, Task::current_user_sp())
}
```

`Signal::do_sigaltstack`: if `oss` non-NULL build `UapiStack` from `task.sas_ss_*` and `on_sig_stack(current_sp)`, then `copy_to_user`. If `ss` is NULL return 0. Else `copy_from_user(ss)`; if `on_sig_stack && !(ss.ss_flags & SS_AUTODISARM)` return `-EPERM`; reject unknown flag bits with `-EINVAL`. If `SS_DISABLE` clear `sas_ss_*` and return 0. Else verify `ss.ss_size >= MINSIGSTKSZ_arch` (`-ENOMEM`), install `sas_ss_sp/size`, `sas_ss_flags = ss.ss_flags & SS_AUTODISARM`. Return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_bits_strict` | INVARIANT | (ss_flags & !(SS_DISABLE|SS_AUTODISARM)) != 0 ⟹ -EINVAL. |
| `min_size_enforced` | INVARIANT | !SS_DISABLE ∧ ss_size < MINSIGSTKSZ ⟹ -ENOMEM. |
| `on_stack_eperm` | INVARIANT | on_sig_stack ∧ ss != NULL ∧ !SS_AUTODISARM ⟹ -EPERM. |
| `oss_reflects_state` | INVARIANT | oss.ss_flags ∈ {SS_DISABLE, SS_ONSTACK, 0} OR'd with SS_AUTODISARM. |
| `disable_clears_all` | INVARIANT | SS_DISABLE ⟹ sas_ss_sp == 0 ∧ sas_ss_size == 0. |
| `per_thread_isolated` | INVARIANT | sibling threads' sas_ss_* unaffected. |

### Layer 2: TLA+

`kernel/sigaltstack.tla`:
- States: COPY_OSS → COPY_SS → VALIDATE_FLAGS → CHECK_ONSTACK → CHECK_SIZE → APPLY.
- Properties:
  - `safety_no_pull_rug` — cannot replace alt-stack while on it without SS_AUTODISARM.
  - `safety_autodisarm_restore` — SS_AUTODISARM: handler-entry clears; sigreturn restores.
  - `safety_per_thread` — per-thread storage; siblings unaffected.
  - `safety_min_size` — installed alt-stack >= MINSIGSTKSZ.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_sigaltstack` post: 0 ⟹ state validly transitioned | `sys_sigaltstack` |
| `do_sigaltstack` post: on_sig_stack ∧ !SS_AUTODISARM ⟹ -EPERM, state unchanged | `do_sigaltstack` |
| `UapiStack::from_user` byte-layout 24 bytes (LP64) | `UapiStack::from_user` |
| `setup_rt_frame` pre: SA_ONSTACK ∧ !on_sig_stack ⟹ uses sas_ss_* | `Signal::setup_rt_frame` |

### Layer 4: Verus / Creusot functional

POSIX `sigaltstack(2)` semantic equivalence. LTP `sigaltstack01..03`, glibc `signal/tst-sigaltstack*`. Go runtime stack-overflow-handler tests.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

- **Per-MINSIGSTKSZ strict** — defense against per-undersized alt-stack overflowing into adjacent VMAs.
- **Per-on-stack EPERM guard** — defense against per-pull-the-rug state corruption.
- **Per-flag-bit strict** — defense against per-future-feature-bit leakage.
- **Per-thread state isolation** — defense against per-thread cross-talk on alt-stack pointers.
- **Per-SS_AUTODISARM atomic save/restore** — defense against per-state leak when a handler on alt stack registers a new alt stack and returns.
- **Per-execve reset** — defense against per-leftover-alt-stack pointing into freed memory after image swap.

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at syscall entry** — randomizes kernel-stack layout per call; combined with per-call `UapiStack` zero-init, defeats per-stack-leak via `stack_t` padding side-channel.
- **PaX UDEREF on `stack_t`** — `copy_from_user(ss)`/`copy_to_user(oss)` run with SMAP/PAN + UDEREF; 24-byte (LP64) read/write bounds-checked.
- **PaX MEMORY_SANITIZE** — local `UapiStack` zeroed; alignment padding between `ss_flags` and `ss_size` cleared — no per-stack/heap byte leak.
- **PaX MPROTECT for alt-stack VMA** — the alt-stack region SHOULD be `mmap(PROT_READ | PROT_WRITE)`; PaX MPROTECT denies subsequent `mprotect(.., PROT_EXEC)` on it, hardening against handler-injection (attacker turning alt-stack into trampoline).
- **PaX VM_LOCKED hint for alt-stack** — userspace SHOULD `mlock()` the alt-stack so it is never paged out; PaX policy via `gradm` can enforce this for SUID/protected subjects. A paged-out alt-stack at SIGSEGV-on-stack-overflow would itself cause recursive SIGSEGV. Kernel does NOT auto-lock (userspace responsibility), but grsec policy can mark the relevant VMA `VM_LOCKED` at handler-install time.
- **GRKERNSEC_SIGNALS sigaltstack audit** — grsec audits (caller-uid, role, ss_sp, ss_size) on every install; suspicious patterns (SUID binary registering alt-stack inside its own .text) trip alert.
- **GRKERNSEC_BRUTE on alt-stack churn** — repeated install/disable cycles by single uid observed by brute-force counter.
- **PaX KERNEXEC** — `do_sigaltstack` in read-only-after-init kernel text; cannot be patched to skip on-stack-EPERM guard or size check.
- **GRKERNSEC_HARDEN_PTRACE** — tracer cannot use `PTRACE_PEEKUSR`/`POKEUSR` to alter `sas_ss_*` across credential boundaries.
- **PaX TASK_HARDENING + RANDSTRUCT** — `sas_ss_sp/size/flags` in randomized layout of `task_struct`, with per-struct slab redzone; OOB writes blocked; field offsets randomized so generic OOB-write primitives cannot deterministically clobber `sas_ss_sp` to redirect signal-frame placement.
- **Per-grsec gradm learning** — RBAC may restrict which subjects can register an alt-stack (e.g. SUID binaries forbidden from registering a W+X region).

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `rt_sigaction(2)` (separate Tier-5 — handler installation incl. SA_ONSTACK).
- Signal-delivery sigframe setup (`setup_rt_frame`, arch sigframe layout — Tier-3).
- Compat 32-bit `compat_sigaltstack` (Tier-3).
- `mmap(MAP_GROWSDOWN)` interaction with alt-stack VMA (Tier-3).
- glibc/musl `SIGSTKSZ` runtime sizing (userspace).
- Implementation code.
