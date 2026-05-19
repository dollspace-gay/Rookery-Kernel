# Tier-5 syscall: prctl(2) — syscall 157

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE5(prctl), prctl_set_mm, ...)
  - include/uapi/linux/prctl.h (PR_SET_*/PR_GET_* option table)
  - kernel/seccomp.c (prctl_set_seccomp)
  - security/commoncap.c (cap_prctl_drop, cap_bset)
  - arch/x86/entry/syscalls/syscall_64.tbl (157  common  prctl)
-->

## Summary

`prctl(2)` is the Swiss-army syscall for per-task / per-thread state knobs: name, capability bounding set, no_new_privs, dumpability, parent-death signal, seccomp filter installation, speculative-execution mitigations, pointer-authentication keys (ARM64), SVE/SME vector-length config, tagged-address mode, THP override, child-subreaper election, MDWE (memory-deny-write-execute), and roughly sixty other options. It is the canonical entry point for sandbox bootstrap (`prctl(PR_SET_NO_NEW_PRIVS, 1)` then `prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, ...)`), for runtime introspection (`prctl(PR_GET_NAME, ...)`), and for arch-neutral hardening toggles.

Critical for: container runtimes, seccomp-loading libraries (libseccomp), language runtimes (Go, glibc, musl) that set thread names, hardening tools (e.g. systemd's `NoNewPrivileges=`, `LockPersonality=`), and ARM64 PAC/MTE/SVE userspace.

This Tier-5 covers the entry surface — option dispatch, argument validation, return contract. The per-option semantics that have their own dedicated syscalls (seccomp, capset) are referenced but not re-specified.

## Signature

```c
long prctl(int option,
           unsigned long arg2,
           unsigned long arg3,
           unsigned long arg4,
           unsigned long arg5);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `option` | `int` | in | One of `PR_*` opcodes (see ABI surface). |
| `arg2..arg5` | `unsigned long` | in/out | Option-dependent; some options write through a user pointer in `arg2`. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Success (option-dependent value, often 0; PR_GET_* options return the queried state). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL`  | Unknown `option`; reserved arg not zero; out-of-range value for option. |
| `EPERM`   | Caller lacks required capability (e.g. `PR_SET_SECUREBITS` needs `CAP_SETPCAP`). |
| `EACCES`  | `PR_SET_SECCOMP` with mode FILTER but `task.no_new_privs == 0` and lacks `CAP_SYS_ADMIN`. |
| `EFAULT`  | User-pointer argument unreadable / unwritable. |
| `EBUSY`   | `PR_SET_MM` while another thread is mutating mm; `PR_SET_VMA` on incompatible vma. |
| `EOPNOTSUPP` | Option not implemented on this architecture (e.g. `PR_PAC_*` on non-ARM64). |
| `ENXIO`   | `PR_MPX_*` on kernels with MPX removed. |

## ABI surface

```text
__NR_prctl (x86_64)   = 157
__NR_prctl (i386)     = 172
__NR_prctl (generic)  = 167   /* arm64, riscv, loongarch */
```

### Option table (PR_SET_* / PR_GET_*)

```text
Identity        1 PR_SET_PDEATHSIG / 2 PR_GET_PDEATHSIG; 3 PR_GET_DUMPABLE / 4 PR_SET_DUMPABLE (0,1,2);
                15 PR_SET_NAME / 16 PR_GET_NAME (TASK_COMM_LEN=16);
                36 PR_SET_CHILD_SUBREAPER / 37 PR_GET_CHILD_SUBREAPER.
CPU/sched       5/6 PR_{GET,SET}_UNALIGN; 7/8 PR_{GET,SET}_KEEPCAPS;
                9/10 PR_{GET,SET}_FPEMU (ia64); 11/12 PR_{GET,SET}_FPEXC (powerpc);
                13/14 PR_{GET,SET}_TIMING; 19/20 PR_{GET,SET}_TSC (PR_TSC_ENABLE|SIGSEGV);
                25/26 PR_{GET,SET}_TIMERSLACK (ns).
Caps/sec        17 PR_GET_SECCOMP / 18 PR_SET_SECCOMP (STRICT(1)|FILTER(2));
                21/22 PR_{GET,SET}_SECUREBITS (needs CAP_SETPCAP);
                23 PR_GET_CAPBSET / 24 PR_CAPBSET_DROP (needs CAP_SETPCAP; irreversible);
                38 PR_GET_NO_NEW_PRIVS / 39 PR_SET_NO_NEW_PRIVS (sticky);
                47 PR_CAP_AMBIENT (OP: RAISE/LOWER/IS_SET/CLEAR_ALL).
Memory          34 PR_MCE_KILL_GET / 35 PR_MCE_KILL; 41/42 PR_{SET,GET}_THP_DISABLE;
                49 PR_SET_VMA (PR_SET_VMA_ANON_NAME, addr, len, name);
                65/66 PR_{SET,GET}_MDWE (PR_MDWE_REFUSE_EXEC_GAIN|NO_INHERIT).
Arch            45/46 PR_{GET,SET}_FP_MODE (mips);
                50/53 PR_{GET,SET}_SPECULATION_CTRL (SSBD/IBPB/L1D_FLUSH; ENABLE/DISABLE/FORCE_DISABLE);
                54 PR_PAC_RESET_KEYS (arm64 PAC: PR_PAC_AP{I,D}{A,B}KEY | APGAKEY);
                55/56 PR_{SET,GET}_IO_FLUSHER; 57 PR_SET_SYSCALL_USER_DISPATCH;
                58/59 PR_PAC_{SET,GET}_ENABLED_KEYS; 62 PR_GET_AUXV.
ARM64           43/44 PR_{SET,GET}_TAGGED_ADDR_CTRL (PR_TAGGED_ADDR_ENABLE | PR_MTE_TCF_* | PR_MTE_TAG_MASK);
                50/51 PR_SVE_{SET,GET}_VL; 60/61 PR_SME_{GET,SET}_VL.
x86 CET         72 PR_GET_SHADOW_STACK_STATUS; 73 PR_SET_SHADOW_STACK_STATUS (ENABLE|WRSS|PUSH);
                74 PR_LOCK_SHADOW_STACK_STATUS (irreversible).
Misc            31 PR_GET_TID_ADDRESS; 48 PR_SET_PTRACER (yama).
```

(Roughly 60 PR_* options total; opcodes are append-only; gaps reserve removed/never-shipped numbers.)

### Compile-time limits

```text
PR_SET_NAME / PR_GET_NAME buffer = TASK_COMM_LEN = 16     /* including NUL */
PR_SET_VMA anon-name           ≤ ANON_VMA_NAME_MAX_LEN = 80
PR_SET_MM   maximum field tag  = PR_SET_MM_MAP_SIZE end
PR_CAP_AMBIENT operations      = 4   (RAISE, LOWER, IS_SET, CLEAR_ALL)
```

## Compatibility contract

REQ-1: Syscall number is **157** on x86_64; **167** on generic-syscall archs. ABI-stable.

REQ-2: Unknown `option` ⟹ `-EINVAL`. The kernel MUST NOT fall through into an unrelated code path. New options append to the end of the table; opcode numbers are append-only.

REQ-3: For each option, unused `argN` arguments MUST be zero or `-EINVAL` is returned (post-v2.6 strictness). Older kernels were tolerant; modern Rookery follows the strict contract.

REQ-4: `PR_SET_NAME`: copies up to 15 bytes from `arg2`, NUL-terminates at position 15. `task.comm` length is fixed `TASK_COMM_LEN = 16`. Sets `current` task's comm; visible via `/proc/self/comm`, `prctl(PR_GET_NAME)`, and `pthread_getname_np`. Per-thread (not per-process).

REQ-5: `PR_SET_NO_NEW_PRIVS`: arg2 MUST be 1; arg3..arg5 MUST be 0. Sets `task.no_new_privs = 1`. **Irreversible** — there is no API to clear it. Inherited by all `fork`/`clone` children. Sticky across `execve`.

REQ-6: `PR_SET_DUMPABLE`: arg2 in `{0, 1, 2}`. `0 = SUID_DUMP_DISABLE`, `1 = SUID_DUMP_USER`, `2 = SUID_DUMP_ROOT` (root-only-readable coredump). Setting to 2 requires `CAP_SYS_PTRACE`. Set to 0 automatically on credential-elevating execve. Affects `/proc/<pid>/{maps,mem,...}` readability.

REQ-7: `PR_SET_PDEATHSIG`: arg2 = signal (0 disables, else 1..NSIG-1). Delivered to the calling task when its real parent dies (not when its reaper changes). Cleared on `execve` (suid/sgid loss) and on `setuid/setgid` that drops privs.

REQ-8: `PR_SET_SECCOMP` with mode FILTER: requires `task.no_new_privs == 1` **or** `CAP_SYS_ADMIN`. arg3 points to a `struct sock_fprog` of length `len <= BPF_MAXINSNS = 4096`. Filter is installed and stacked atop any existing filters (LIFO evaluation).

REQ-9: `PR_CAPBSET_DROP`: arg2 = cap (0..CAP_LAST_CAP). Removes cap from `task.cap_bset`. Requires `CAP_SETPCAP`. **Irreversible** — no API to raise capbset.

REQ-10: `PR_CAP_AMBIENT`: arg2 = OP, arg3 = cap. RAISE requires cap to be in both `permitted` and `inheritable`. LOWER and CLEAR_ALL are unconditional. IS_SET returns 0/1.

REQ-11: `PR_SET_KEEPCAPS`: arg2 in {0,1}. When 1, permitted caps are preserved across a `setuid(nonzero)` call that would otherwise drop them. Cleared automatically on credential-elevating execve.

REQ-12: `PR_SET_THP_DISABLE`: arg2 in {0,1}. Inherited by children via fork; cleared at execve.

REQ-13: `PR_SET_VMA(PR_SET_VMA_ANON_NAME, addr, len, name)`: name string ≤ `ANON_VMA_NAME_MAX_LEN = 80`, allowed chars `[A-Za-z0-9_.+-]`. Applies to anonymous vmas in `[addr, addr+len)`. Visible in `/proc/<pid>/maps` (`[anon:<name>]`).

REQ-14: `PR_SET_MDWE`: arg2 mask of `PR_MDWE_REFUSE_EXEC_GAIN | PR_MDWE_NO_INHERIT`. **Irreversible per process**: once set, future `mmap(PROT_EXEC | PROT_WRITE)` and `mprotect` that grows the X set fails `-EACCES`. NO_INHERIT controls fork propagation.

REQ-15: `PR_GET_TSC` / `PR_SET_TSC` (x86): PR_TSC_ENABLE (default) or PR_TSC_SIGSEGV (rdtsc raises SIGSEGV in user mode).

REQ-16: `PR_SET_SPECULATION_CTRL`: arg2 = feature, arg3 = state. FORCE_DISABLE is irreversible per-task.

REQ-17: ARM64 ops: `PR_PAC_RESET_KEYS` regenerates PAC keys (existing signed pointers invalidated); `PR_SET_TAGGED_ADDR_CTRL` enables TBI + MTE-tag-check-fault mode; `PR_SVE_SET_VL` sets vector length, returns actual ≤ requested, `PR_SVE_VL_INHERIT` controls execve propagation.

REQ-18: `PR_GET_AUXV`: copies up to arg3 bytes of auxv to arg2; returns full auxv size (may exceed arg3).

REQ-19: `PR_SET_SYSCALL_USER_DISPATCH`: arg5 points to a user byte; when nonzero, syscalls outside [arg3, arg3+arg4) deliver `SIGSYS`.

REQ-20: Arch-only options on wrong arch return `-EINVAL` (Rookery preserves legacy semantics).

## Acceptance Criteria

- [ ] AC-1: `PR_SET_NAME`/`PR_GET_NAME` roundtrip (truncated to 15).
- [ ] AC-2: `PR_SET_NO_NEW_PRIVS, 1` succeeds; no API clears it.
- [ ] AC-3: After NNP, execve of setuid binary preserves euid.
- [ ] AC-4: `PR_SET_DUMPABLE, 0` makes `/proc/self/maps` unreadable by other uids.
- [ ] AC-5: `PR_SET_DUMPABLE, 2` without `CAP_SYS_PTRACE` returns `-EPERM`.
- [ ] AC-6: `PR_SET_PDEATHSIG, SIGTERM`: parent death delivers SIGTERM to caller.
- [ ] AC-7: `PR_SET_SECCOMP, FILTER` without NNP/`CAP_SYS_ADMIN` returns `-EACCES`.
- [ ] AC-8: `PR_CAPBSET_DROP` without `CAP_SETPCAP` returns `-EPERM`.
- [ ] AC-9: After `PR_CAPBSET_DROP(CAP_SYS_ADMIN)`, `PR_GET_CAPBSET` returns 0 for it.
- [ ] AC-10: `PR_CAP_AMBIENT_RAISE` for cap not in permitted+inheritable returns `-EPERM`.
- [ ] AC-11: `PR_SET_KEEPCAPS, 1` then `setuid(1000)` preserves permitted caps.
- [ ] AC-12: `PR_SET_THP_DISABLE, 1` disables THP for current and future mappings.
- [ ] AC-13: `PR_SET_VMA, PR_SET_VMA_ANON_NAME, addr, len, "myheap"` shows `[anon:myheap]` in `/proc/self/maps`.
- [ ] AC-14: `PR_SET_MDWE, PR_MDWE_REFUSE_EXEC_GAIN` then `mmap(PROT_WRITE|PROT_EXEC)` returns `-EACCES`.
- [ ] AC-15: `PR_SET_TSC, PR_TSC_SIGSEGV`: `rdtsc` delivers SIGSEGV.
- [ ] AC-16: `PR_PAC_RESET_KEYS` on non-ARM64 returns `-EINVAL`.
- [ ] AC-17: Unknown option returns `-EINVAL`.
- [ ] AC-18: `PR_GET_AUXV, buf, 4096`: returns full auxv size; writes up to 4096 bytes.
- [ ] AC-19: `PR_SET_SPECULATION_CTRL, ..., PR_SPEC_FORCE_DISABLE` then re-enable returns `-EPERM`.
- [ ] AC-20: `PR_SET_CHILD_SUBREAPER, 1` makes caller the reaper for orphaned descendants.

## Architecture

```rust
#[syscall(nr = 157, abi = "sysv")]
pub fn sys_prctl(
    option: i32,
    arg2:   usize,
    arg3:   usize,
    arg4:   usize,
    arg5:   usize,
) -> isize {
    Prctl::dispatch(option, arg2, arg3, arg4, arg5)
}
```

`Prctl::dispatch(option, a2, a3, a4, a5) -> isize`:
1. match option {
2.   PR_SET_PDEATHSIG / PR_GET_PDEATHSIG / PR_SET_DUMPABLE / PR_GET_DUMPABLE   => Prctl::pdeath_dumpable(...),
3.   PR_SET_NAME / PR_GET_NAME                                                  => Prctl::name(...),
4.   PR_SET_NO_NEW_PRIVS / PR_GET_NO_NEW_PRIVS                                  => Prctl::nnp(...),
5.   PR_SET_SECCOMP / PR_GET_SECCOMP                                            => Seccomp::via_prctl(...),
6.   PR_SET_SECUREBITS / PR_GET_SECUREBITS / PR_CAPBSET_DROP / PR_GET_CAPBSET / PR_CAP_AMBIENT / PR_SET_KEEPCAPS / PR_GET_KEEPCAPS => Cap::op(...),
7.   PR_SET_THP_DISABLE / PR_GET_THP_DISABLE / PR_SET_VMA / PR_SET_MDWE / PR_GET_MDWE => Mm::op(...),
8.   PR_SET_TSC / PR_GET_TSC / PR_SET_SPECULATION_CTRL / PR_GET_SPECULATION_CTRL => Arch::op(...),
9.   PR_PAC_RESET_KEYS / PR_SVE_SET_VL / PR_SVE_GET_VL / PR_SET_TAGGED_ADDR_CTRL => Arm64::op(...),
10.  PR_SET_PTRACER                                                             => Yama::set_ptracer(a2 as i32),
11.  PR_SET_CHILD_SUBREAPER / PR_GET_AUXV                                       => Prctl::misc(...),
12.  _                                                                           => Err(EINVAL),
13. }

`Prctl::set_nnp(a2, a3, a4, a5)`:
1. if a2 != 1 || a3 != 0 || a4 != 0 || a5 != 0 { return Err(EINVAL); }
2. /* irreversible, sticky */
3. task.no_new_privs.store(true, Ordering::SeqCst);
4. Ok(0)

`Prctl::set_dumpable(a2)`:
1. if a2 > SUID_DUMP_ROOT { return Err(EINVAL); }
2. if a2 == SUID_DUMP_ROOT && !capable(CAP_SYS_PTRACE) { return Err(EPERM); }
3. task.dumpable = a2 as u32;
4. Ok(0)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `option_validated` | INVARIANT | unknown option ⟹ early `-EINVAL` before any state mutation. |
| `nnp_irreversible` | INVARIANT | `task.no_new_privs` ∈ {0→1}; never 1→0. |
| `capbset_monotone_drop` | INVARIANT | `task.cap_bset` only loses bits; never gains via prctl. |
| `name_buffer_bounded` | INVARIANT | PR_SET_NAME copy bounded by TASK_COMM_LEN = 16. |
| `dumpable_range_checked` | INVARIANT | PR_SET_DUMPABLE arg2 ∈ {0,1,2}. |
| `seccomp_filter_needs_nnp_or_cap` | INVARIANT | PR_SET_SECCOMP FILTER requires NNP ∨ CAP_SYS_ADMIN. |
| `mdwe_irreversible` | INVARIANT | `task.mm.flags.mdwe` ∈ {0→nonzero}; never reset. |
| `spec_ctrl_force_disable_sticky` | INVARIANT | SPEC_FORCE_DISABLE ⟹ no re-enable possible. |

### Layer 2: TLA+

`kernel/prctl.tla`:
- States per task: {comm, dumpable, no_new_privs, cap_bset, ambient, securebits, mdwe, thp_disable, tsc_mode, spec_ctrl}.
- Properties:
  - `safety_unknown_option_eperm_or_einval` — unknown opcodes return `-EINVAL`; never partial mutation.
  - `safety_nnp_sticky` — once 1, always 1.
  - `safety_capbset_monotone` — never re-add.
  - `safety_mdwe_monotone` — never clear.
  - `safety_dispatch_total` — every option in opcode table reachable.
  - `liveness_returns` — every call terminates with a finite return.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `dispatch` post: option not in table ⟹ Err(EINVAL) | `Prctl::dispatch` |
| `set_name` post: task.comm ASCII NUL-terminated, len ≤ 15 | `Prctl::set_name` |
| `set_nnp` post: task.no_new_privs == 1 | `Prctl::set_nnp` |
| `set_dumpable` post: task.dumpable ∈ {0,1,2} | `Prctl::set_dumpable` |
| `capbset_drop` post: cap removed from bset; never added | `Cap::bset_drop` |

### Layer 4: Verus / Creusot functional

Per-`prctl(2)` man-page semantic equivalence. LTP `prctl0[1-9]` and the seccomp-prctl interaction tests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`prctl(2)` reinforcement:

- **Per-`PR_SET_NO_NEW_PRIVS` irreversible** — defense against per-sandbox-bypass via post-exec privilege gain.
- **Per-`PR_SET_DUMPABLE = 0` enforced under suid** — defense against per-coredump-credential-leak after setuid.
- **Per-`PR_CAPBSET_DROP` monotone** — defense against per-capability re-raise from compromised user code.
- **Per-`PR_SET_MDWE` irreversible** — defense against per-W^X bypass via runtime mprotect.
- **Per-`PR_SET_SECCOMP` requires NNP or `CAP_SYS_ADMIN`** — defense against per-filter-bypass via privilege escalation.
- **Per-`PR_SET_SECUREBITS` requires `CAP_SETPCAP`** — defense against per-securebits flip from unprivileged.
- **Per-`PR_SET_PDEATHSIG` cleared on suid-exec** — defense against per-parent-spoof signal injection.
- **Per-`PR_SET_SPECULATION_CTRL` FORCE_DISABLE sticky** — defense against per-side-channel re-enable.

## Grsecurity / PaX surface

- **PaX UDEREF on `PR_SET_NAME`/`PR_GET_NAME` user buffer** — defense against per-comm-buffer kernel deref.
- **PaX UDEREF on `PR_SET_VMA` anon name copy** — defense against per-anon-name kernel read.
- **PAX_RANDKSTACK at prctl entry** — randomizes kernel stack offset per invocation.
- **GRKERNSEC_HARDEN_PTRACE interaction** — `PR_SET_PTRACER` (yama) further constrained by grsec process-isolation policy.
- **GRKERNSEC_CHROOT_CAPS** — inside chroot, `PR_CAPBSET_DROP` cannot re-grant; entry into chroot already strips `CAP_SETPCAP`/`CAP_SETUID`/`CAP_SETGID`.
- **GRKERNSEC: PR_SET_NO_NEW_PRIVS mandatory for sandboxes** — sandbox-launcher policy refuses to install seccomp without NNP precondition.
- **GRKERNSEC: PR_SET_DUMPABLE = 0 enforced on every suid execve** — kernel forces dumpable=0 regardless of prctl.
- **PaX SEGMEXEC / MPROTECT** — `PR_SET_MDWE` integrates with the PaX MPROTECT lock so W^X cannot be relaxed once enforced.
- **Per-PR_SET_SECCOMP grsec audit** — every filter installation logged with task creds and call site.
- **Per-PR_SET_VMA name sanitized** — grsec rejects names with control chars or path-like sequences that could spoof /proc/maps output.
- **Per-PR_SET_SPECULATION_CTRL** — grsec policy can pin all tasks to FORCE_DISABLE on boot, ignoring per-task re-enable attempts.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `seccomp(2)` syscall (Tier-5 separate doc).
- `capset(2)`/`capget(2)` (Tier-5 separate docs).
- Full PaX MPROTECT integration semantics (Tier-3 in `mm/pax-mprotect.md`).
- Yama LSM internals (`security/yama.md` Tier-3).
- Implementation code.
