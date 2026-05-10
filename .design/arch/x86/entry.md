# Tier-3: arch/x86/entry — syscall, IRQ, and exception entry

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - arch/x86/entry/
  - arch/x86/entry/entry_64.S
  - arch/x86/entry/entry_64_compat.S
  - arch/x86/entry/entry_64_fred.S
  - arch/x86/entry/entry_fred.c
  - arch/x86/entry/syscall_64.c
  - arch/x86/entry/syscall_32.c
  - arch/x86/entry/syscalls/syscall_64.tbl
  - arch/x86/entry/syscalls/syscall_32.tbl
  - arch/x86/entry/calling.h
  - arch/x86/entry/thunk.S
  - arch/x86/kernel/traps.c
  - arch/x86/kernel/idt.c
  - arch/x86/kernel/irq.c
  - arch/x86/kernel/irq_64.c
  - arch/x86/kernel/irqinit.c
  - arch/x86/kernel/nmi.c
  - arch/x86/kernel/signal.c
  - arch/x86/kernel/signal_64.c
  - arch/x86/kernel/signal_32.c
  - arch/x86/mm/cpu_entry_area.c
  - arch/x86/mm/fault.c
  - arch/x86/include/asm/ptrace.h
  - arch/x86/include/asm/desc.h
  - arch/x86/include/uapi/asm/sigcontext.h
  - arch/x86/include/uapi/asm/ucontext.h
-->

## Summary
Tier-3 design for x86_64 kernel entry: every CPU-mode transition from user → kernel and back. Owns the syscall path (`syscall`/`int 0x80`/`sysenter`/FRED), exception delivery (page fault, GP, UD, DE, OF, etc. → signals), IRQ entry/exit (hardirq + softirq dispatch), NMI handling, the cpu_entry_area read-only mapping, signal-frame construction + sigreturn, the entry-side per-task kernel-stack switch, and the `pt_regs` register save/restore convention. Owns the implementation of MPROTECT-W→X-block per the locked `00-security-principles.md` decision (the prctl is here; mmap-side enforcement is in `mm/mmap.md`).

This Tier-3 owns one of the densest concurrency surfaces in the kernel: every CPU at every moment is either in entry, exit, or user-mode; getting it wrong corrupts every running process.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Classic IDT-based entry | `arch/x86/entry/entry_64.S`, `arch/x86/entry/entry_64_compat.S`, `arch/x86/entry/calling.h`, `arch/x86/entry/thunk.S` |
| FRED entry (Flexible Return and Event Delivery — newer Intel) | `arch/x86/entry/entry_64_fred.S`, `arch/x86/entry/entry_fred.c` |
| Syscall dispatch tables | `arch/x86/entry/syscall_64.c`, `arch/x86/entry/syscall_32.c`, `arch/x86/entry/syscalls/syscall_{64,32}.tbl` |
| Exception handlers (high-level C) | `arch/x86/kernel/traps.c`, `arch/x86/mm/fault.c` (page-fault handler) |
| IDT setup | `arch/x86/kernel/idt.c`, `arch/x86/include/asm/desc.h` |
| IRQ entry / dispatch | `arch/x86/kernel/irq.c`, `arch/x86/kernel/irq_64.c`, `arch/x86/kernel/irqinit.c` |
| NMI | `arch/x86/kernel/nmi.c`, `arch/x86/kernel/hw_breakpoint.c` |
| Signal-frame construction + sigreturn | `arch/x86/kernel/signal.c`, `arch/x86/kernel/signal_64.c`, `arch/x86/kernel/signal_32.c` |
| Userspace-visible signal/ptrace structs | `arch/x86/include/uapi/asm/{sigcontext,ucontext,ptrace}.h`, `arch/x86/include/asm/ptrace.h` |
| cpu_entry_area (per-CPU read-only mapping for entry) | `arch/x86/mm/cpu_entry_area.c` |
| 32-bit compat (ia32) | `arch/x86/ia32/`, `arch/x86/entry/entry_64_compat.S`, `arch/x86/entry/syscalls/syscall_32.tbl` |
| FPU save/restore | `arch/x86/kernel/fpu/` (core.c, init.c, regset.c, xstate.c, signal.c, bugs.c) |
| Userspace ELF launcher | `arch/x86/kernel/process_64.c` (cross-ref `fs/exec-binfmt.md`) |

## Compatibility contract

`arch/x86/00-overview.md` enumerated the syscall ABI; this Tier-3 owns the byte-identity. Highlights:

### Syscall instruction handling

- `syscall` (the preferred 64-bit instruction; saves RIP→RCX, RFLAGS→R11, jumps to `IA32_LSTAR`) — owned here
- `int 0x80` (legacy; userspace via interrupt vector 0x80) — owned here for compat
- `sysenter` (32-bit fast-syscall) — owned here for ia32 compat layer
- FRED entry (alternative to `syscall` on supporting CPUs) — both paths supported per `arch/x86/00-overview.md` D2

### Register convention (x86_64 syscall)

| Direction | Register usage |
|---|---|
| In | `rax` = syscall number; `rdi`, `rsi`, `rdx`, `r10`, `r8`, `r9` = args 0..5 |
| Out | `rax` = result (negative `[-4095, -1]` = `-errno`) |
| Clobbered | `rcx` (saved RIP), `r11` (saved RFLAGS) |
| Preserved | All other GPRs and segment registers |
| FPU/SIMD | Preserved across syscall by lazy save/restore |

### `pt_regs` layout

`arch/x86/include/asm/ptrace.h` defines `struct pt_regs`. ABI-visible via:
- `ptrace(PTRACE_GETREGS, ...)` returns a `struct user_regs_struct` byte-identical to upstream
- ELF core dump `NT_PRSTATUS` note contains `pt_regs`
- `/proc/<pid>/stat` exposes some pt_regs fields via the formatted line

Field layout per upstream: `r15, r14, r13, r12, bp, bx, r11, r10, r9, r8, ax, cx, dx, si, di, orig_ax, ip, cs, flags, sp, ss` (21 fields × 8 bytes = 168 bytes).

### Signal-frame layout

`arch/x86/include/uapi/asm/sigcontext.h` defines `struct sigcontext` (kernel saves to user stack on signal delivery; `sigreturn` reads back). Layout MUST be byte-identical:

- `r8` through `r15` (8 × 8 bytes)
- `rdi`, `rsi`, `rbp`, `rbx`, `rdx`, `rax`, `rcx`, `rsp`, `rip`, `eflags` (10 × 8 bytes)
- `cs`, `gs`, `fs`, `__pad0` (4 × 2 bytes)
- `err`, `trapno`, `oldmask`, `cr2` (4 × 8 bytes)
- `fpstate` pointer (8 bytes)
- 8 reserved 8-byte words

The user-visible signal frame includes a `struct ucontext_t` (`include/uapi/asm/ucontext.h`) wrapping the `sigcontext` plus the signal stack pointer and FPU state.

### Exception → signal mapping

| Exception (vector) | Mnemonic | Signal | siginfo `si_code` |
|---|---|---|---|
| 0 | #DE (divide error) | SIGFPE | FPE_INTDIV |
| 3 | #BP (breakpoint) | SIGTRAP | TRAP_BRKPT |
| 4 | #OF (overflow) | SIGSEGV | SEGV_MAPERR |
| 5 | #BR (bound range exceeded) | SIGSEGV | SEGV_BNDERR |
| 6 | #UD (invalid opcode) | SIGILL | ILL_ILLOPC |
| 7 | #NM (device not available) | SIGFPE | FPE_FLTINV |
| 13 | #GP (general protection) | SIGSEGV | SEGV_MAPERR |
| 14 | #PF (page fault) | SIGSEGV / SIGBUS | SEGV_MAPERR / SEGV_ACCERR |
| 16 | #MF (x87 FP error) | SIGFPE | FPE_FLT* |
| 17 | #AC (alignment check) | SIGBUS | BUS_ADRALN |
| 18 | #MC (machine check) | SIGBUS / panic | (implementation-defined) |
| 19 | #XF (SIMD FP error) | SIGFPE | FPE_FLT* |

Mapping is byte-identical to upstream; userspace signal-handler test suites depend on exact `si_code` values.

### IDT layout

`arch/x86/kernel/idt.c` populates the 256-entry IDT at boot. Vector 0..31 are CPU exceptions (per Intel SDM). Vector 32..255 are kernel-defined: 32..47 are legacy ISA IRQs, 48..238 are device IRQs (LAPIC-vectored), 239..255 are special (LAPIC error, spurious, NMI shortcut, etc.).

The IDT is per-CPU on x86_64 (each CPU has its own; the BSP IDT is the canonical reference). Layout matches upstream so ptrace-driven introspection of `idt_table` symbol matches upstream.

### MPROTECT-W→X-block + NOEXEC-strict (the JIT exemption locus)

Per `00-security-principles.md` Locked default-policy table, MPROTECT-W→X-block defaults ON system-wide. Implementation split:

- **prctl `PR_REQUEST_EXEC_GAIN`** is owned by `kernel/00-overview.md` § task-lifecycle.md (the syscall handler + per-task state). Cross-ref.
- **ELF note recognition** (`NT_ROOKERY_SECURITY_FLAGS`) is owned by `fs/00-overview.md` § exec-binfmt.md. Cross-ref.
- **Enforcement at `mmap`/`mprotect`** is owned by `mm/00-overview.md` § mmap.md. Cross-ref.

This Tier-3 owns the **entry-time per-task state propagation** — when a task transitions user→kernel, the `exec_gain_state` field of `task_struct` is read; when kernel→user, no propagation needed. This Tier-3 has the per-task state enum:

```rust
#[repr(u8)]
pub enum ExecGainState {
    Disabled = 0,         // Default: cannot mprotect W→X or mmap with PROT_EXEC|PROT_WRITE
    NeedsExecGain = 1,    // ELF note bit 0 OR PR_REQUEST_EXEC_GAIN bit 0 set
    NeedsRwxAnon = 2,     // ELF note bit 1 OR PR_REQUEST_EXEC_GAIN bit 1 set (additive over NeedsExecGain)
}
```

## Requirements

- REQ-1: Every x86_64 syscall in `arch/x86/entry/syscalls/syscall_64.tbl` (421 entries at baseline) reaches its handler with byte-identical register state to upstream.
- REQ-2: 32-bit-on-64-bit compat syscalls (`int 0x80`, `sysenter`, `syscall32`) per `syscall_32.tbl` reach their handlers with byte-identical register state.
- REQ-3: FRED entry path (CONFIG_X86_FRED) coexists with classic IDT-based entry on FRED-capable CPUs; both produce identical observable state.
- REQ-4: `pt_regs` layout is byte-identical; `ptrace(PTRACE_GETREGS)` and ELF `NT_PRSTATUS` produce byte-identical buffers vs. upstream.
- REQ-5: Signal-frame construction (`struct sigcontext` + `struct ucontext_t`) is byte-identical; `sigreturn(2)` consumes the frame identically; `sigaltstack(2)` switches signal stacks identically.
- REQ-6: All 32 CPU-exception → signal mappings are byte-identical (vector → signo + si_code per the table above).
- REQ-7: IDT layout is identical: vectors 0..31 fixed by CPU, 32..255 per upstream's `idt_init_data` array; per-CPU IDT setup matches upstream.
- REQ-8: Per-task `exec_gain_state` is read on every user→kernel transition; mprotect/mmap consumers (cross-ref `mm/mmap.md`) honor it; `clone(2)` inherits, `execve(2)` resets per ELF note.
- REQ-9: NMI handler preserves `pt_regs` correctly even when interrupting another exception or syscall path; nested-NMI handling matches upstream.
- REQ-10: cpu_entry_area is per-CPU, mapped read-only into kernel space; per-CPU IDT, GDT, TSS, IST stacks all mapped here; layout matches upstream so kdb / drgn / crash can introspect.
- REQ-11: FPU/SIMD state preservation across syscall + signal: lazy save on first FPU use after kernel entry; XSAVE/XRSTOR matches upstream `xstate` feature set per CPUID.
- REQ-12: Spectre-v2 mitigations (retpoline + IBRS + IBPB + eIBRS where supported) applied at entry/exit boundaries per `arch/x86/cpu-mitigations.md` (Tier 3 to be authored). Default-on per `00-security-principles.md`.
- REQ-13: RANDKSTACK (random kernel-stack offset on syscall entry) default-on per `00-security-principles.md`. Implementation: at syscall-entry, after saving caller pt_regs and switching to the kernel stack, advance RSP by a per-syscall-random small offset.
- REQ-14: Per-task kernel-stack switch on entry: every entry from userspace switches RSP from user stack to per-task kernel stack via the per-CPU TSS->`rsp0` field. Per-task kernel stacks are isolated (PRIVATE_KSTACKS mandate per `00-security-principles.md`).
- REQ-15: vsyscall page (legacy fixed mapping at `0xffffffffff600000`) operates in EMULATE mode by default per `arch/x86/00-overview.md` D5; vector 0x14 of trap handler emulates the legacy syscalls.
- REQ-16: SMAP/SMEP CR4 bits set during early boot (`arch/x86/boot.md`) are NOT modified at runtime; entry/exit relies on them for kernel↔user separation (CLOSE_KERNEL/CLOSE_USERLAND mandate per `00-security-principles.md`).
- REQ-17: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A `strace` golden-trace of every syscall in `syscall_64.tbl` shows byte-identical entry/exit register conventions vs. upstream. (covers REQ-1)
- [ ] AC-2: A 32-bit `i386` userspace test program issuing each `int 0x80` and `syscall32` produces identical observable behavior. (covers REQ-2)
- [ ] AC-3: Boot Rookery on FRED-capable hardware (`-cpu max,+fred` in qemu); confirm syscalls work; same for FRED-incapable (`-cpu max,-fred`). (covers REQ-3)
- [ ] AC-4: `ptrace`-based test reads `NT_PRSTATUS` from a tracee; `pahole struct pt_regs` matches upstream. (covers REQ-4)
- [ ] AC-5: A signal-handler test that prints `mcontext_t` field-by-field shows byte-identical layout vs. upstream. `sigaltstack` round-trip succeeds. (covers REQ-5)
- [ ] AC-6: For each CPU exception 0..19, a userspace test triggers it and asserts the (signal, si_code) pair matches upstream. (covers REQ-6)
- [ ] AC-7: A diff between Rookery's and upstream's IDT contents (visible via kdb or drgn `idt_table`) is empty. (covers REQ-7)
- [ ] AC-8: A JIT runtime built without the exec-gain ELF note fails to mprotect W→X (returns EACCES); after `prctl(PR_REQUEST_EXEC_GAIN, 1, ...)` it succeeds. With the ELF note set at build time, no prctl needed. (covers REQ-8)
- [ ] AC-9: A test injects an NMI mid-syscall; the kernel state remains coherent; `dmesg` shows correct stack trace. (covers REQ-9)
- [ ] AC-10: `pahole cpu_entry_area` produces byte-identical layout vs. upstream. (covers REQ-10)
- [ ] AC-11: An FPU test using AVX-512 across a signal handler returns identical state on Rookery vs. upstream. (covers REQ-11)
- [ ] AC-12: `/sys/devices/system/cpu/vulnerabilities/spectre_v2` shows the same mitigation strings as upstream on the same CPU. (covers REQ-12)
- [ ] AC-13: A statistical test (1000 syscalls, RSP recorded) shows kernel-stack offset randomization on Rookery; same test on upstream with `randomize_kstack_offset=1` shows equivalent distribution. (covers REQ-13)
- [ ] AC-14: A test allocates `nr_threads` of identical workload; each thread's kernel-stack is at a distinct address (per-task isolation). (covers REQ-14)
- [ ] AC-15: An old binary using `time(2)` via vsyscall page (constructable with `-fno-builtin time` plus a `0xffffffffff600400` literal) executes correctly on Rookery's EMULATE mode. (covers REQ-15)
- [ ] AC-16: SMAP/SMEP-disabled boot (`clearcpuid=smap,smep`) refuses to bring up entry path; SMAP/SMEP-enabled boot succeeds. (covers REQ-16)
- [ ] AC-17: Hardening section is present and follows template. (covers REQ-17)

## Architecture

### Rust module organization

- `kernel::arch::x86::entry::syscall` — syscall dispatch (calls into the cross-arch syscall handler in `kernel/syscall-entry-helpers.md`)
- `kernel::arch::x86::entry::exception` — exception handlers; each maps to a signal per the table above
- `kernel::arch::x86::entry::irq` — IRQ entry; dispatches to the registered handler list (cross-ref `kernel/irq.md`)
- `kernel::arch::x86::entry::nmi` — NMI entry; nested-NMI handling
- `kernel::arch::x86::entry::fred` — FRED-mode entry; conditionally compiled
- `kernel::arch::x86::idt` — IDT setup + per-CPU IDT
- `kernel::arch::x86::pt_regs` — `pt_regs` definition (`#[repr(C)]`, byte-identical to upstream)
- `kernel::arch::x86::signal` — signal-frame construction + sigreturn handler
- `kernel::arch::x86::cpu_entry_area` — per-CPU entry-mapping data structure
- `kernel::arch::x86::fpu` — FPU state save/restore (XSAVE/XRSTOR)
- `kernel::arch::x86::randkstack` — per-syscall kernel-stack offset randomization

Assembly portions remain `.S` files: `entry_64.S`, `entry_64_compat.S`, `entry_64_fred.S`, `thunk.S`, `calling.h` (header for asm macros). They live at the upstream paths, built via kbuild.

### Entry path (classic IDT, x86_64 syscall)

```
[user-mode] syscall instruction
   ↓ CPU loads RIP from IA32_LSTAR; switches CS+SS via IA32_STAR; RIP→RCX, RFLAGS→R11
[entry_64.S::entry_SYSCALL_64]
   ↓ saves user pt_regs to kernel stack (switched via TSS->rsp0)
   ↓ applies RANDKSTACK offset (REQ-13)
   ↓ enters Rust dispatcher: kernel::arch::x86::entry::syscall::dispatch(pt_regs)
   ↓ Rust dispatcher reads rax for syscall number; consults kernel::arch::x86::entry::SYSCALL_TABLE
   ↓ calls the cross-arch syscall handler
[handler returns Result<i64, Error>]
   ↓ Rust converts to rax (negative = -errno)
   ↓ entry_64.S::sysret_check_kernel_state — verifies no kernel state leak
   ↓ sysretq instruction returns to userspace
```

### Entry path (FRED, when CONFIG_X86_FRED + capable CPU)

```
[user-mode] syscall (or any other event)
   ↓ FRED hardware flow: stack switch, CR3 swap, RIP load — much less assembly
[entry_64_fred.S::entry_FRED_call0]
   ↓ enters entry_fred.c::fred_handle_syscall
   ↓ Rust dispatcher (same as classic)
[returns]
   ↓ FRED ERETU instruction
```

### Signal delivery flow

```
[Task is in user mode; signal pending]
   ↓ next user→kernel transition (any) sets TIF_SIGPENDING handler in entry C path
[arch_do_signal_or_restart in arch/x86/kernel/signal.c]
   ↓ saves user pt_regs to user stack as struct sigcontext + ucontext_t
   ↓ sets up signal-handler call frame
   ↓ sigreturn restorer = vDSO::__kernel_rt_sigreturn
[returns to user signal handler]
   ↓ handler runs
   ↓ handler returns to vDSO::__kernel_rt_sigreturn
[that calls rt_sigreturn syscall]
   ↓ entry path → kernel::arch::x86::signal::sigreturn(pt_regs)
   ↓ restores user pt_regs from user stack
   ↓ returns to original interrupted point
```

### IRQ entry

Per-vector IDT entries point at small assembly stubs in `entry_64.S` that:
1. Save partial `pt_regs`
2. Switch to per-CPU IRQ stack (via TSS->ist[]) for hardware IRQs; stay on per-task stack for software interrupts
3. Call into Rust handler `kernel::arch::x86::entry::irq::handle_irq(vector, pt_regs)`
4. On exit, check pending softirqs and dispatch via `do_softirq`
5. Return

### NMI handling

NMIs can interrupt anything (including other NMIs). Path:
1. NMI vector (2) entry stub
2. Detects nested NMI via per-CPU `nmi_state` flag
3. Saves partial pt_regs
4. Calls `kernel::arch::x86::entry::nmi::handle_nmi(pt_regs)`
5. Returns via `iretq`

### Locking and concurrency

Entry path runs without kernel locks (all state is per-CPU or per-task). Exception/IRQ entry disables interrupts on entry, re-enables after switching to kernel stack. Soft-irq dispatch on exit uses `local_bh_disable`/`enable`.

NMI is the most concurrency-sensitive: nested-NMI uses lock-free state machine via per-CPU atomic. TLA+ model (`models/arch/x86/nmi_nesting.tla`, added in this Tier-3) proves nested-NMI invariants.

### Error handling

Entry-path errors translate to signal delivery (REQ-6); never `Result`-typed return. The Rust-side `EntryOutcome` enum:

```rust
pub enum EntryOutcome {
    Continue,                                  // resume userspace
    DeliverSignal { sig: Signal, info: SigInfo, // queue signal for next user-return
    BugOn(BugReason),                          // unrecoverable; kernel::panic!
}
```

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `pt_regs` field reads on entry (assumes hardware contract) | `kani::proofs::arch::x86::entry::pt_regs_read_safety` |
| Signal-frame construction (writes to user stack) | `kani::proofs::arch::x86::signal::frame_safety` |
| sigreturn (reads from user stack) | `kani::proofs::arch::x86::signal::sigreturn_safety` |
| IDT table writes | `kani::proofs::arch::x86::idt::install_safety` |
| Nested NMI state machine | `kani::proofs::arch::x86::entry::nmi_state_safety` |
| FPU XSAVE/XRSTOR | `kani::proofs::arch::x86::fpu::xstate_safety` |
| RANDKSTACK offset application | `kani::proofs::arch::x86::randkstack::offset_safety` |

### Layer 2: TLA+ models

- `models/arch/x86/idt_install.tla` (inherited from `arch/x86/00-overview.md` Layer-2 mandates) — proves IDT entry installation is atomic w.r.t. concurrent interrupt delivery.
- `models/arch/x86/nmi_nesting.tla` (NEW in this Tier-3) — proves nested-NMI state machine: every NMI is acknowledged exactly once; no NMI is lost; an NMI interrupting another NMI's handler doesn't corrupt state.
- `models/arch/x86/exec_gain_propagation.tla` (NEW) — proves per-task `exec_gain_state` propagates correctly across `clone`/`execve`/`fork`.

### Layer 3: Kani harnesses for data-structure invariants

- `idt_table` integrity (per `arch/x86/00-overview.md` Layer-3 mandates) — every vector 0–31 maps to its CPU-fixed handler kind; every vector 32–255 is either unset or maps to a registered IRQ
- `cpu_entry_area` layout (per `arch/x86/00-overview.md` Layer-3 mandates) — TSS IST stacks within the per-CPU entry-mapping area
- `pt_regs` field offsets — match upstream byte-for-byte (a layout-only invariant; checked at compile time via const assertions)

### Layer 4: Functional correctness (opt-in)

- **Exception → signal mapping**: Verus proof that for any (vector, error_code, cr2) tuple that the CPU emits, the kernel emits exactly one (signal, si_code) per the table above. Tractable; high-leverage (a single-table parser).

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **PRIVATE_KSTACKS** | Per-task kernel stack switched in via TSS->rsp0; per-CPU IST stacks for NMI/MCE/double-fault | `00-security-principles.md` § Mandatory |
| **RANDKSTACK** | Per-syscall stack-offset randomization via `kernel::arch::x86::randkstack` (sysctl `kernel.randomize_kstack_offset=1` flipped on by default) | `00-security-principles.md` § Default-on configurable off |
| **CLOSE_KERNEL / CLOSE_USERLAND** (SMEP/SMAP gates) | Entry/exit relies on CR4 SMEP+SMAP set during boot; no runtime modification | `00-security-principles.md` § Mandatory |
| **DIRECT_CALL** (Spectre-v2 entry-side mitigation) | Retpolined indirect calls in entry asm + IBRS write on entry / clear on exit (per CPU policy in cpu-mitigations.md) | `00-security-principles.md` § Default-on configurable off |
| **MPROTECT-W→X-block** (per-task entry-time state propagation) | Reads `task->exec_gain_state` on every user→kernel transition; the actual mmap/mprotect enforcement lives in `mm/00-overview.md` § mmap.md | `00-security-principles.md` § Default-on system-wide with per-process exemption |

### Row-1 features consumed by this component

- **KERNEXEC**: entry assembly is in `.text` (RX); the entry path never writes to itself
- **UDEREF**: `pt_regs` from userspace is treated as untrusted; access via `UserPtr<...>` newtype; kernel-side `pt_regs` is `&mut`
- **AUTOSLAB**: pt_regs is on the kernel stack, not slab-allocated, but signal-frame structures use type-tagged caches when allocated
- **SIZE_OVERFLOW**: stack-pointer arithmetic in entry uses `checked_add`/`wrapping_add` per the lint; all pt_regs offsets use named constants

### Row-2 / GR-RBAC integration

Entry path runs before any LSM hook is reached. The first LSM-relevant hook in any syscall path is the syscall-specific entry hook (`security_*` functions) which lives in the cross-arch dispatcher. This component has NO LSM-hook surface of its own.

### Userspace-visible behavior changes

Per Axiom 4 of `00-security-principles.md`:
- **MPROTECT-W→X-block**: default-on; per-process exemption via ELF note + prctl. JIT runtimes without the note observe `EACCES` from `mprotect(PROT_EXEC|PROT_WRITE)`. Documented in the user manual + distro packagers' migration guide.
- **NOEXEC-strict**: same exemption mechanism. JITs without the note cannot mmap with PROT_EXEC over anonymous regions.
- **RANDKSTACK**: kernel-internal; no userspace observability. (Strictly speaking, an extremely careful timing-side-channel attack might detect the randomization; this is a feature, not a bug.)

### Verification

(See § Verification above; consolidated table covers Layers 1–4.)

## Open Questions

<!-- OPEN: Q1 -->
### Q1: Per-syscall TLA+ model coverage
There are 421 x86_64 syscalls. Some have nontrivial entry-path concurrency contracts (e.g., `clone3` interacts with task lifecycle; `rt_sigreturn` writes per-task signal state under signal-pending lock). Should each get a Layer-2 TLA+ model, or only the locking-heavy ones?

**Recommendation**: Only locking-heavy syscalls. Most syscalls are pass-through dispatchers; their concurrency is owned by the called subsystem. Models stay with the subsystem (mm/, fs/, kernel/, net/) per their Layer-2 declarations. The entry path itself owns only `nmi_nesting`, `idt_install`, and `exec_gain_propagation` models.

**To resolve**: User confirms.
<!-- /OPEN -->

## Out of Scope

- 32-bit kernel entry (`X86_32=y`) — out per `arch/x86/00-overview.md` D1
- vsyscall NONE / XONLY modes — only EMULATE supported in v0 per D5
- Implementation code
