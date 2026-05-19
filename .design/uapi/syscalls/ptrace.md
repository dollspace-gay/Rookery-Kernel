# Tier-5 syscall: ptrace(2) — syscall 101

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/ptrace.c (SYSCALL_DEFINE4(ptrace), ptrace_attach, ptrace_request)
  - include/uapi/linux/ptrace.h (PTRACE_*, PTRACE_O_*, struct ptrace_peeksiginfo_args)
  - arch/x86/kernel/ptrace.c (arch_ptrace, regset access)
  - security/yama/yama_lsm.c (Yama ptrace scope)
  - arch/x86/entry/syscalls/syscall_64.tbl (101  common  ptrace)
-->

## Summary

`ptrace(2)` is the kernel's debugging primitive. A "tracer" task attaches to a "tracee" task and acquires the ability to read and write its registers and memory, intercept its syscalls, single-step it, stop and resume it, observe its signals, and follow it across fork/clone/exec. Linux supports two attach modes: the legacy `PTRACE_ATTACH` (sends `SIGSTOP`) and the modern `PTRACE_SEIZE` (no SIGSTOP, supports interrupt and detailed options). Every debugger (gdb, lldb, strace), every sandbox supervisor (`PTRACE_SYSCALL`, `PTRACE_O_TRACESECCOMP`), and every crash-dump tool (`gcore`) is built on this syscall.

ptrace is also the historical attack surface — credential leaks across attach, races at fork, signal injection. Yama LSM clamps the cross-process attach by default; grsecurity goes further with `GRKERNSEC_HARDEN_PTRACE` (full process isolation).

This Tier-5 covers the syscall entry; the per-request architecture-specific regset/regs/breakpoint machinery is Tier-3 in `arch/x86/ptrace.md`.

## Signature

```c
long ptrace(enum __ptrace_request request,
            pid_t pid,
            void *addr,
            void *data);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `request` | `enum` | in | One of `PTRACE_*` opcodes (~50 total). |
| `pid` | `pid_t` | in | Target tid (TID, not TGID, except `PTRACE_TRACEME` ignores this). |
| `addr` | `void *` | in/out | Request-specific. For PEEK/POKE, the tracee's userspace address. For GETREGSET, an `iovec*` or NT_* constant. |
| `data` | `void *` | in/out | Request-specific. For POKE, the value. For GETREGS, a `struct user_regs_struct *`. For CONT, a signal number. |

## Return value

| Value | Meaning |
|---|---|
| `0` (or peeked value) | Success. `PTRACE_PEEK*` returns the peeked word; ambiguous-with-error: use `errno = 0` before call. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM`   | LSM/Yama/grsec denied attach; tracee is dumpable=0 and tracer lacks `CAP_SYS_PTRACE`; cross-uid without cap; tracee is init / kernel thread. |
| `ESRCH`   | `pid` does not exist; tracee not stopped when request requires stopped state; not tracing. |
| `EIO`     | `addr` invalid in tracee mm (legacy PEEK/POKE); request invalid for tracee state. |
| `EINVAL`  | Unknown `request`; bad `addr` alignment; unsupported regset NT_* on this arch. |
| `EBUSY`   | Tracee already traced; tracee in zombie state for some requests. |
| `EFAULT`  | User buffer in tracer's mm unreadable/unwritable. |

## ABI surface

```text
__NR_ptrace (x86_64)   = 101
__NR_ptrace (i386)     = 26
__NR_ptrace (generic)  = 117      /* arm64, riscv */

Attach/detach
  PTRACE_TRACEME              0    /* tracee: "trace me, my parent is now tracer" */
  PTRACE_ATTACH               16   /* tracer: attach + send SIGSTOP */
  PTRACE_SEIZE                0x4206 /* tracer: attach without SIGSTOP; options in data */
  PTRACE_DETACH               17   /* detach; data = signal to deliver */
  PTRACE_INTERRUPT            0x4207 /* SEIZE-only: cause tracee to stop now */
  PTRACE_KILL                 8    /* deprecated; sends SIGKILL */

Stop control
  PTRACE_CONT                 7    /* resume; data = signal to inject */
  PTRACE_SINGLESTEP           9    /* execute one insn then stop */
  PTRACE_SYSCALL              24   /* resume until next syscall entry/exit */
  PTRACE_SYSEMU               31   /* x86: trap before syscall executes, allow emulation */
  PTRACE_SYSEMU_SINGLESTEP    32
  PTRACE_LISTEN               0x4208 /* SEIZE-only: tracee remains group-stopped */

Memory access
  PTRACE_PEEKTEXT             1    /* read word from tracee mm; address in addr; word returned */
  PTRACE_PEEKDATA             2    /* synonym */
  PTRACE_POKETEXT             4    /* write word; addr = tracee addr; data = value */
  PTRACE_POKEDATA             5
  PTRACE_PEEKSIGINFO          0x4209 /* iterate pending siginfo */
  PTRACE_GET_SYSCALL_INFO     0x420e /* struct ptrace_syscall_info */

Register access
  PTRACE_GETREGS              12   /* fills user_regs_struct */
  PTRACE_SETREGS              13
  PTRACE_GETFPREGS            14
  PTRACE_SETFPREGS            15
  PTRACE_GETFPXREGS           18   /* x86 extended */
  PTRACE_SETFPXREGS           19
  PTRACE_GETREGSET            0x4204 /* addr = NT_* constant; data = iovec */
  PTRACE_SETREGSET            0x4205

Options
  PTRACE_SETOPTIONS           0x4200 /* data = PTRACE_O_* mask */
  PTRACE_GETEVENTMSG          0x4201 /* data = unsigned long *event_msg */
  PTRACE_GETSIGINFO           0x4202
  PTRACE_SETSIGINFO           0x4203
  PTRACE_GETSIGMASK           0x420a
  PTRACE_SETSIGMASK           0x420b
  PTRACE_SECCOMP_GET_FILTER   0x420c /* dump filter image */
  PTRACE_SECCOMP_GET_METADATA 0x420d

PTRACE_O_* (set via PTRACE_SETOPTIONS or PTRACE_SEIZE data)
  PTRACE_O_TRACESYSGOOD        1 << 0    /* OR 0x80 into stop signo for syscall stops */
  PTRACE_O_TRACEFORK           1 << 1
  PTRACE_O_TRACEVFORK          1 << 2
  PTRACE_O_TRACECLONE          1 << 3
  PTRACE_O_TRACEEXEC           1 << 4
  PTRACE_O_TRACEVFORKDONE      1 << 5
  PTRACE_O_TRACEEXIT           1 << 6
  PTRACE_O_TRACESECCOMP        1 << 7
  PTRACE_O_EXITKILL            1 << 20   /* SIGKILL tracee if tracer dies */
  PTRACE_O_SUSPEND_SECCOMP     1 << 21   /* CAP_SYS_ADMIN-only; bypass tracee's seccomp filters */

PTRACE_EVENT_* (in WIFSTOPPED stop signal >> 16)
  PTRACE_EVENT_FORK            1
  PTRACE_EVENT_VFORK           2
  PTRACE_EVENT_CLONE           3
  PTRACE_EVENT_EXEC            4
  PTRACE_EVENT_VFORK_DONE      5
  PTRACE_EVENT_EXIT            6
  PTRACE_EVENT_SECCOMP         7
  PTRACE_EVENT_STOP            128       /* SEIZE-initiated stop, not SIGSTOP */
```

## Compatibility contract

REQ-1: Syscall number is **101** on x86_64; **117** on generic-syscall archs. ABI-stable.

REQ-2: `PTRACE_TRACEME`: tracee calls; its parent becomes tracer; pid/addr/data ignored.

REQ-3: `PTRACE_ATTACH`: tracer calls with target tid; sends SIGSTOP; reparented for ptrace purposes only; `-EPERM` on LSM/Yama/grsec deny.

REQ-4: `PTRACE_SEIZE`: like ATTACH but no SIGSTOP; `data` = initial PTRACE_O_* options; tracer uses PTRACE_INTERRUPT to stop.

REQ-5: `PTRACE_DETACH`: ends tracing; `data = signo` to deliver on resume (0 for none).

REQ-6: Attach-time cred check: tracer EUID == tracee EUID AND tracee dumpable == 1 AND no LSM denies (yama default: ancestor-or-CAP_SYS_PTRACE) — OR tracer has `CAP_SYS_PTRACE` in tracee's user-ns.

REQ-7: `PTRACE_CONT`/`SINGLESTEP`/`SYSCALL`/`SYSEMU`: tracee must be ptrace-stopped; `data = signo` to inject (or 0).

REQ-8: `PEEKTEXT/PEEKDATA`: returns long-sized word from tracee mm at addr (unaligned OK). Caller sets `errno = 0` pre-call to disambiguate -1 word vs error. `POKETEXT/POKEDATA`: writes `data` as long at addr.

REQ-9: `GETREGS`/`SETREGS`: per-arch `struct user_regs_struct`. `GETREGSET(NT_*, iovec*)`: arch-neutral; NT_PRSTATUS=general, NT_PRFPREG=fp, NT_X86_XSTATE=XSAVE, NT_ARM_SVE=SVE, NT_ARM_PAC_MASK=PAC keys.

REQ-10: `SETOPTIONS`: `data = PTRACE_O_*` mask; unrecognized bits → `-EINVAL`. `O_EXITKILL`: tracer dies ⟹ tracee SIGKILL'd.

REQ-14: `PTRACE_O_SUSPEND_SECCOMP` requires `CAP_SYS_ADMIN`; bypasses tracee seccomp during single-step. `PTRACE_O_TRACESECCOMP`: SECCOMP_RET_TRACE triggers `PTRACE_EVENT_SECCOMP` stop.

REQ-15: Group-stop tracee visible via wait4; resume via PTRACE_CONT. `PTRACE_GET_SYSCALL_INFO` writes `struct ptrace_syscall_info` (op/arch/IP/SP/entry-or-exit details).

REQ-16: At most one tracer per task; re-attach → `-EPERM`. `PTRACE_TRACEME` once per task lifetime.

REQ-17: Tracing across cred transitions: if tracee execve's a suid binary AND tracer lacks `CAP_SYS_PTRACE`, binary loaded NON-suid (AT_SECURE=0, file caps stripped) — prevents escalation via tracer mem-poke.

REQ-18: `PTRACE_PEEKSIGINFO`: dumps pending signals; flags = PTRACE_PEEKSIGINFO_SHARED for tgid-shared.

REQ-19: Yama `kernel.yama.ptrace_scope`: 0 = classic uid match; 1 (default) = ancestor or `PR_SET_PTRACER`; 2 = only `CAP_SYS_PTRACE`; 3 = disabled.

## Acceptance Criteria

- [ ] AC-1: `PTRACE_TRACEME` from child + parent wait4: child stopped at next execve with SIGTRAP.
- [ ] AC-2: `PTRACE_ATTACH` to sibling: tracee receives SIGSTOP; tracer sees via wait4.
- [ ] AC-3: `PTRACE_SEIZE`: no SIGSTOP; tracer can `PTRACE_INTERRUPT` to stop.
- [ ] AC-4: Cross-uid attach without `CAP_SYS_PTRACE` returns `-EPERM`.
- [ ] AC-5: Attach to dumpable=0 task without `CAP_SYS_PTRACE` returns `-EPERM`.
- [ ] AC-6: `PEEKDATA`/`POKEDATA` read/write tracee mm word.
- [ ] AC-7: `GETREGS`/`GETREGSET(NT_PRSTATUS)` populate regs.
- [ ] AC-8: `CONT` injects signal in data arg.
- [ ] AC-9: `SYSCALL` triggers stop at next syscall entry.
- [ ] AC-10: `SETOPTIONS(PTRACE_O_TRACEEXEC)` then tracee execve delivers `PTRACE_EVENT_EXEC`.
- [ ] AC-11: `PTRACE_O_EXITKILL` + tracer exits: tracee SIGKILL'd.
- [ ] AC-12: Attach to already-traced returns `-EPERM`.
- [ ] AC-13: `DETACH` then PEEKDATA returns `-ESRCH`.
- [ ] AC-14: yama.ptrace_scope=2: attach without `CAP_SYS_PTRACE` → `-EPERM`.
- [ ] AC-15: yama.ptrace_scope=3: all attaches → `-EPERM`.
- [ ] AC-16: `SUSPEND_SECCOMP` without `CAP_SYS_ADMIN` → `-EPERM`.
- [ ] AC-17: Tracee execve of suid binary under non-CAP_SYS_PTRACE tracer: euid==ruid.
- [ ] AC-18: `GET_SYSCALL_INFO` reports op=ENTRY at enter-stop, op=EXIT at exit-stop.

## Architecture

```rust
#[syscall(nr = 101, abi = "sysv")]
pub fn sys_ptrace(request: i64, pid: i32, addr: usize, data: usize) -> isize {
    Ptrace::dispatch(request, pid, addr, data)
}
```

`Ptrace::dispatch(request, pid, addr, data) -> isize`:
1. if request == PTRACE_TRACEME { return Ptrace::traceme(); }
2. let tracee = Task::find_by_tid(pid).ok_or(ESRCH)?;
3. if request == PTRACE_ATTACH || request == PTRACE_SEIZE {
4.   return Ptrace::attach(tracee, request, data);
5. }
6. /* All other requests require tracer relationship + stopped (for most) */
7. if tracee.ptrace_tracer != Some(current()) { return Err(ESRCH); }
8. match request {
9.   DETACH/INTERRUPT/CONT/SYSCALL/SINGLESTEP                   => Ptrace::resume_family(tracee, request, data),
10.  PEEKTEXT/PEEKDATA/POKETEXT/POKEDATA                        => Ptrace::peek_poke(tracee, request, addr, data),
11.  GETREGS/SETREGS/GETREGSET/SETREGSET                        => Ptrace::regs(tracee, request, addr, data),
12.  SETOPTIONS/GETEVENTMSG/GETSIGINFO/SETSIGINFO/GET_SYSCALL_INFO => Ptrace::info(tracee, request, addr, data),
13.  _                                                          => Arch::ptrace(tracee, request, addr, data),
14. }

`Ptrace::attach(tracee, request, options)`:
1. security::ptrace_access_check(tracee, PTRACE_MODE_ATTACH)?;   // LSM
2. yama::ptrace_check(tracee)?;                                  // Yama
3. grsec::ptrace_check(tracee)?;                                 // Grsec
4. if tracee.ptrace_tracer.is_some() { return Err(EPERM); }
5. tracee.ptrace_tracer = Some(current().tid);
6. if request == PTRACE_ATTACH { signal::send_to(tracee, SIGSTOP); }
7. if request == PTRACE_SEIZE { Ptrace::set_options(tracee, options as u32)?; }
8. Ok(0)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `single_tracer` | INVARIANT | tracee.tracer ∈ {None, Some(tid)}; attach-while-traced ⟹ `-EPERM`. |
| `cred_check_at_attach` | INVARIANT | attach validates LSM/yama/grsec before any state mutation. |
| `request_requires_stopped` | INVARIANT | PEEK/POKE/GETREGS require tracee in PTRACE-stopped state. |
| `nnp_blocks_suid_via_tracer` | INVARIANT | traced execve of suid: euid unchanged when tracer lacks CAP_SYS_PTRACE. |
| `option_mask_validated` | INVARIANT | SETOPTIONS data & ~KNOWN_O == 0. |
| `traceme_once` | INVARIANT | task may set TRACEME at most once per lifetime. |

### Layer 2: TLA+

`kernel/ptrace.tla`:
- States per task: {tracer: Option<tid>, stop_state, options, pending_siginfo}.
- Properties:
  - `safety_single_tracer` — invariant.
  - `safety_cred_check_first` — attach state-mutation only after creds OK.
  - `safety_detach_atomic` — DETACH transitions tracer → None atomically.
  - `safety_suid_traced` — traced exec of setuid binary preserves caller creds.
  - `liveness_attached_eventually_stops` — ATTACH delivers SIGSTOP and tracee enters stop within bounded time.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `attach` post: tracee.tracer == Some(current) ∨ error | `Ptrace::attach` |
| `detach` post: tracee.tracer == None | `Ptrace::detach` |
| `peek` post: returned word matches tracee mm at addr | `Ptrace::peek` |
| `set_options` post: tracee.ptrace_options == data (only known bits) | `Ptrace::set_options` |

### Layer 4: Verus / Creusot functional

Per-`ptrace(2)` man-page semantic equivalence. LTP `ptrace01..ptrace12`, kselftest `ptrace_*` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`ptrace(2)` reinforcement:

- **Per-tracer single ownership** — defense against per-double-attach race.
- **Per-LSM/yama/grsec triple check at attach** — defense against per-cross-uid attach.
- **Per-dumpable=0 attach denial** — defense against per-suid-mem-poke escalation.
- **Per-suid-binary credential strip when traced** — defense against per-tracer-mem-poke privilege escalation.
- **Per-O_EXITKILL** — defense against per-orphaned-tracee continuing without supervisor.
- **Per-O_SUSPEND_SECCOMP `CAP_SYS_ADMIN` gate** — defense against per-seccomp-bypass via debugger.
- **Per-request stopped-state check** — defense against per-race-window mem corruption.

## Grsecurity / PaX surface

- **GRKERNSEC_HARDEN_PTRACE (process-isolation)** — ALL cross-process ptrace requires `CAP_SYS_PTRACE` (stricter than yama=2); ancestor-only path disabled.
- **PaX UDEREF on tracer's user buffers** — `data`, addr-pointed user data, iovec content all accessed via UDEREF/PAN.
- **PAX_RANDKSTACK at ptrace entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_CHROOT_CAPS** — inside chroot, ptrace caps stripped (effective `-EPERM`).
- **GRKERNSEC_PROC restrictions** — `/proc/<tid>/{maps,mem,environ}` of dumpable=0 hidden from non-CAP_SYS_PTRACE readers.
- **Per-grsec audit on every ATTACH/SEIZE** — tracer creds, tracee creds, opcode, PC logged.
- **PaX RAP** — POKE writes into tracee text checked against per-tracee RAP function-type hashes.
- **Per-grsec suid-traced strip** — even CAP_SYS_PTRACE tracers cause traced suid execve to strip privs (stricter than upstream).
- **Per-grsec PTRACE_O_SUSPEND_SECCOMP refused** — even with CAP_SYS_ADMIN; grsec treats as undefined-behavior.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Architecture-specific regset semantics (Tier-3 in `arch/x86/ptrace.md`, `arch/arm64/ptrace.md`).
- Yama LSM internals (Tier-3 in `security/yama.md`).
- Group-stop / SIGSTOP semantics (Tier-3 in `kernel/signal.md`).
- ptrace-seccomp event protocol (Tier-3 in `kernel/seccomp.md`).
- Implementation code.
