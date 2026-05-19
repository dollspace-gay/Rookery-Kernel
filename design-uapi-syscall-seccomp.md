---
title: "Tier-5 syscall: seccomp(2) — syscall 317"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`seccomp(2)` is the explicit, future-proof entry point for installing seccomp filters and querying seccomp state. It supersedes the legacy `prctl(PR_SET_SECCOMP, ...)` interface by accepting a `flags` argument and supporting operations that prctl could not express (user-notify FD, sync across threads, log policy). Once a filter is installed, every subsequent syscall in the task is filtered: each filter (a BPF program) is run in LIFO order over a `struct seccomp_data` describing the syscall; the most restrictive action wins.

Critical for: container runtimes (Docker, podman, kubernetes), browsers (Chrome, Firefox), systemd's `SystemCallFilter=`, sandboxes (firejail, bubblewrap), and security-critical daemons (sshd's pre-auth privsep).

This Tier-5 covers the syscall entry surface; the BPF filter execution machinery (cBPF -> eBPF lowering, JIT, evaluation order) is Tier-3 in `kernel/seccomp.c`.

### Acceptance Criteria

- [ ] AC-1: `seccomp(SECCOMP_SET_MODE_STRICT, 0, NULL)` succeeds; subsequent `getpid()` delivers SIGKILL.
- [ ] AC-2: `seccomp(SECCOMP_SET_MODE_FILTER, 0, &fprog)` without NNP and without `CAP_SYS_ADMIN` returns `-EACCES`.
- [ ] AC-3: After `prctl(PR_SET_NO_NEW_PRIVS, 1)`, `seccomp(SECCOMP_SET_MODE_FILTER, 0, &allow_all_fprog)` succeeds.
- [ ] AC-4: Filter with `len = 0` returns `-EINVAL`; filter with `len = 4097` returns `-EINVAL`.
- [ ] AC-5: Filter returning `SECCOMP_RET_ERRNO | EPERM` for `getpid` causes `getpid()` to return `-EPERM`.
- [ ] AC-6: Filter returning `SECCOMP_RET_KILL_PROCESS` for `getpid` kills the entire process with SIGSYS.
- [ ] AC-7: Filter returning `SECCOMP_RET_TRAP | 42` for `getpid` delivers SIGSYS with `si_code = SYS_SECCOMP`, `si_errno = 42`.
- [ ] AC-8: Two filters: first deny `read` with `ERRNO`, second deny `read` with `KILL_THREAD`; result is KILL_THREAD (most restrictive).
- [ ] AC-9: `SECCOMP_FILTER_FLAG_TSYNC` propagates filter to all threads in tgid.
- [ ] AC-10: `SECCOMP_FILTER_FLAG_NEW_LISTENER | SECCOMP_FILTER_FLAG_TSYNC` returns `-EINVAL`.
- [ ] AC-11: `SECCOMP_FILTER_FLAG_NEW_LISTENER` returns a positive fd; filter returning `USER_NOTIF` posts an event there.
- [ ] AC-12: `GET_ACTION_AVAIL` with USER_NOTIF returns 0; with 0xdeadbeef returns `-EOPNOTSUPP`.
- [ ] AC-13: `GET_NOTIF_SIZES` writes nonzero values for all three sizes.
- [ ] AC-14: Filters preserved across `fork` and `execve`.
- [ ] AC-15: Installing > 64 filters returns `-EINVAL`.
- [ ] AC-16: After SET_MODE_FILTER via `CAP_SYS_ADMIN`, `task.no_new_privs == 1`.

### Architecture

```rust
#[syscall(nr = 317, abi = "sysv")]
pub fn sys_seccomp(operation: u32, flags: u32, args: UserPtr<u8>) -> isize {
    Seccomp::dispatch(operation, flags, args)
}
```

`Seccomp::dispatch(op, flags, args) -> isize`:
1. match op {
2.   SECCOMP_SET_MODE_STRICT     => Seccomp::set_mode_strict(flags, args),
3.   SECCOMP_SET_MODE_FILTER     => Seccomp::set_mode_filter(flags, args),
4.   SECCOMP_GET_ACTION_AVAIL    => Seccomp::get_action_avail(flags, args),
5.   SECCOMP_GET_NOTIF_SIZES     => Seccomp::get_notif_sizes(flags, args),
6.   _                           => Err(EINVAL),
7. }

`Seccomp::set_mode_filter(flags, args)`:
1. /* Validate flags */
2. let known = SECCOMP_FILTER_FLAG_TSYNC | _LOG | _SPEC_ALLOW | _NEW_LISTENER | _TSYNC_ESRCH | _WAIT_KILLABLE_RECV;
3. if flags & !known != 0 { return Err(EINVAL); }
4. if (flags & TSYNC) && (flags & NEW_LISTENER) { return Err(EINVAL); }
5. /* Privilege gate */
6. if !task.no_new_privs && !ns_capable(CAP_SYS_ADMIN) { return Err(EACCES); }
7. /* Copy + verify program */
8. let fprog: SockFprog = UserPtr::copy(args)?;
9. if fprog.len == 0 || fprog.len > BPF_MAXINSNS { return Err(EINVAL); }
10. let prog = Bpf::cbpf_to_ebpf_verify(fprog)?;
11. /* Install */
12. let filter = Filter::new(prog, flags);
13. if task.seccomp.filter_chain_len() >= MAX_FILTERS_PER_TASK { return Err(EINVAL); }
14. task.seccomp.push(filter);
15. if !task.no_new_privs { task.no_new_privs.store(true, SeqCst); }   // sticky NNP
16. /* Optional TSYNC */
17. if flags & TSYNC { Seccomp::tsync(filter, flags)?; }
18. /* Optional NEW_LISTENER */
19. if flags & NEW_LISTENER {
20.   let fd = Seccomp::install_listener_fd(filter)?;
21.   return Ok(fd as isize);
22. }
23. Ok(0)

`Seccomp::set_mode_strict(flags, args)`: flags == 0 && args.is_null() && mode == DISABLED required; sets mode = STRICT.

`Seccomp::run_filter_chain(data) -> Action`: iterate filters in LIFO; result = min(all filter outputs); smallest numeric action wins (KILL_PROCESS most restrictive).

### Out of Scope

- cBPF/eBPF verifier (Tier-3 in `kernel/bpf/verifier.md`).
- Filter JIT (Tier-3 in `arch/x86/bpf-jit.md`).
- User-notify (`SECCOMP_IOCTL_NOTIF_*`) ioctls (Tier-5 in `uapi/syscalls/ioctl-seccomp-notif.md`).
- `/proc/sys/kernel/seccomp/*` sysctls (Tier-5 in `uapi/sysctl/seccomp.md`).
- Implementation code.

### signature

```c
int seccomp(unsigned int operation,
            unsigned int flags,
            void *args);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `operation` | `unsigned int` | in | `SECCOMP_SET_MODE_STRICT` / `SECCOMP_SET_MODE_FILTER` / `SECCOMP_GET_ACTION_AVAIL` / `SECCOMP_GET_NOTIF_SIZES`. |
| `flags` | `unsigned int` | in | Operation-specific bitmask (`SECCOMP_FILTER_FLAG_*`); 0 for ops that don't take flags. |
| `args` | `void *` | in/out | Operation-specific pointer (e.g. `struct sock_fprog *` for SET_MODE_FILTER). |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Success. For SET_MODE_FILTER with `TSYNC | NEW_LISTENER`, returns the new notify FD. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL`   | Unknown `operation`; unknown flag; filter length > `BPF_MAXINSNS`; mode-strict and a thread is multi-threaded with non-trivial filter. |
| `EFAULT`   | `args` or filter program pointer unreadable. |
| `EACCES`   | `SECCOMP_SET_MODE_FILTER` and `task.no_new_privs == 0` and lacks `CAP_SYS_ADMIN` in the task's user-namespace. |
| `ENOMEM`   | Filter allocation failed. |
| `ENOSYS`   | seccomp not compiled in (`!CONFIG_SECCOMP`). |
| `EBUSY`    | `TSYNC` requested but another thread refused (returned tid in error path). |
| `ESRCH`    | Notify-fd ID mismatch in respond ioctl (separate from this entry). |
| `EOPNOTSUPP` | `SECCOMP_GET_ACTION_AVAIL` for an unrecognized action value. |

### abi surface

```text
__NR_seccomp (x86_64)  = 317
__NR_seccomp (i386)    = 354
__NR_seccomp (generic) = 277      /* arm64, riscv */

Operations
  SECCOMP_SET_MODE_STRICT       0   /* args ignored; allows only read/write/_exit/sigreturn */
  SECCOMP_SET_MODE_FILTER       1   /* args = struct sock_fprog *; installs filter */
  SECCOMP_GET_ACTION_AVAIL      2   /* args = u32 *action; returns 0 if available, else -EOPNOTSUPP */
  SECCOMP_GET_NOTIF_SIZES       3   /* args = struct seccomp_notif_sizes *; reports sizeof for user-notify ABI */

Flags (for SECCOMP_SET_MODE_FILTER)
  SECCOMP_FILTER_FLAG_TSYNC                  (1 << 0)  /* sync filter to all threads in tgid */
  SECCOMP_FILTER_FLAG_LOG                    (1 << 1)  /* log action even if not RET_LOG */
  SECCOMP_FILTER_FLAG_SPEC_ALLOW             (1 << 2)  /* skip SSBD mitigation for filtered task */
  SECCOMP_FILTER_FLAG_NEW_LISTENER           (1 << 3)  /* return a notify-fd */
  SECCOMP_FILTER_FLAG_TSYNC_ESRCH            (1 << 4)  /* on TSYNC failure return -ESRCH (not tid) */
  SECCOMP_FILTER_FLAG_WAIT_KILLABLE_RECV     (1 << 5)  /* user-notify recv waits killably */

Filter return values (cBPF/eBPF program output, 32-bit "action | data")
  SECCOMP_RET_KILL_PROCESS  0x80000000U
  SECCOMP_RET_KILL_THREAD   0x00000000U   /* historical alias SECCOMP_RET_KILL */
  SECCOMP_RET_TRAP          0x00030000U
  SECCOMP_RET_ERRNO         0x00050000U   /* low 16 bits = errno */
  SECCOMP_RET_USER_NOTIF    0x7fc00000U
  SECCOMP_RET_TRACE         0x7ff00000U
  SECCOMP_RET_LOG           0x7ffc0000U
  SECCOMP_RET_ALLOW         0x7fff0000U

struct sock_fprog {
    unsigned short      len;          /* number of BPF insns; <= BPF_MAXINSNS */
    struct sock_filter *filter;       /* user-space pointer */
};

struct seccomp_data {                  /* read-only view passed to the BPF program */
    int       nr;                      /* syscall number */
    __u32     arch;                    /* AUDIT_ARCH_* */
    __u64     instruction_pointer;     /* PC at syscall */
    __u64     args[6];                 /* a0..a5 */
};
```

### Compile-time limits

```text
BPF_MAXINSNS              = 4096
SECCOMP_MAX_FILTER_LENGTH = BPF_MAXINSNS
MAX_FILTERS_PER_TASK      = 64        /* installed-filter chain depth */
NOTIF_RING_DEPTH          = 1024      /* outstanding user-notify events */
```

### compatibility contract

REQ-1: Syscall number is **317** on x86_64; **277** on generic-syscall archs. ABI-stable forever.

REQ-2: `SECCOMP_SET_MODE_STRICT`: `flags` MUST be 0; `args` MUST be `NULL`. Sets `task.seccomp.mode = SECCOMP_MODE_STRICT`. Subsequent syscalls limited to `read`, `write`, `_exit`, `sigreturn`; any other syscall delivers SIGKILL. Cannot be combined with FILTER.

REQ-3: `SECCOMP_SET_MODE_FILTER`: `args = struct sock_fprog *`. Requires `task.no_new_privs == 1` OR `ns_capable(user_ns, CAP_SYS_ADMIN)`. Otherwise `-EACCES`.

REQ-4: Filter length `sock_fprog.len` MUST satisfy `0 < len <= BPF_MAXINSNS = 4096`. Else `-EINVAL`.

REQ-5: Filter is verified by the cBPF (classic BPF) verifier and converted to internal eBPF program (or JITed). Verification failure → `-EINVAL`.

REQ-6: Filters stack LIFO: the most-recently-installed filter runs FIRST on each syscall. The action returned is the *highest-priority* (smallest numeric value) action across ALL filters: KILL_PROCESS < KILL_THREAD < TRAP < ERRNO < USER_NOTIF < TRACE < LOG < ALLOW.

REQ-7: Max installed filters per task = `MAX_FILTERS_PER_TASK = 64`. Beyond this `-EINVAL`.

REQ-8: `SECCOMP_FILTER_FLAG_TSYNC`: after install, attempt to copy filter set to every other thread in `task.tgid`. On any thread refusing (different mode or different filter set), revert and return `-EBUSY` (or `-ESRCH` if `TSYNC_ESRCH` set), with the offending tid in the return value if `!TSYNC_ESRCH`.

REQ-9: `SECCOMP_FILTER_FLAG_LOG`: each ALLOW result is logged via audit subsystem. Per-action logging policy in `/proc/sys/kernel/seccomp/actions_logged`.

REQ-10: `SECCOMP_FILTER_FLAG_NEW_LISTENER`: returns a new file descriptor (the "notify fd") on success. The fd is used by a supervisor process to receive `seccomp_notif` events when filters return `RET_USER_NOTIF` and respond via `SECCOMP_IOCTL_NOTIF_*`. Only one listener per filter; combining with TSYNC returns `-EINVAL`.

REQ-11: `SECCOMP_FILTER_FLAG_SPEC_ALLOW`: requests skipping SSBD speculation mitigation for the task. Requires `CAP_SYS_ADMIN`.

REQ-12: `SECCOMP_GET_ACTION_AVAIL`: `args = const __u32 *action`. Returns 0 if action is supported by the kernel, `-EOPNOTSUPP` if not.

REQ-13: `SECCOMP_GET_NOTIF_SIZES`: `args = struct seccomp_notif_sizes *`. Writes the kernel's current sizes for `struct seccomp_notif`, `struct seccomp_notif_resp`, `struct seccomp_data`. Enables forward-ABI for user-notify.

REQ-14: Filter set is inherited by `fork`/`clone` (children get the parent's chain). Filters are also preserved across `execve` (this is the entire point of seccomp).

REQ-15: Once a filter is installed, `task.no_new_privs` is **forced to 1** (in case it was not set when CAP_SYS_ADMIN path was used).

REQ-16: Filter actions: `KILL_PROCESS` kills the tgid with `SIGSYS`; `KILL_THREAD` kills caller; `ERRNO` returns `-data` (low 16 bits, clamped to `MAX_ERRNO = 4095`); `TRAP` delivers `SIGSYS` with `si_code = SYS_SECCOMP`, `si_errno = data`; `USER_NOTIF` blocks and posts to listener fd; `ALLOW` proceeds.

REQ-17: `struct seccomp_data` is read-only; filters see pointer values only (no pointee deref). Filters MUST gate on `arch` field (else cross-arch syscall confusion).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `op_validated` | INVARIANT | unknown operation ⟹ `-EINVAL`. |
| `flags_validated` | INVARIANT | flags & ~KNOWN == 0. |
| `len_bounded` | INVARIANT | 0 < fprog.len ≤ BPF_MAXINSNS. |
| `nnp_or_cap_for_filter` | INVARIANT | SET_MODE_FILTER ⟹ NNP ∨ CAP_SYS_ADMIN. |
| `filter_chain_bounded` | INVARIANT | chain_len ≤ MAX_FILTERS_PER_TASK. |
| `nnp_set_after_filter` | INVARIANT | post-install: task.no_new_privs == 1. |
| `action_ordering` | INVARIANT | run_chain returns min(actions); KILL_PROCESS dominates. |

### Layer 2: TLA+

`kernel/seccomp.tla`:
- States per task: {mode, filters: List, no_new_privs}.
- Properties:
  - `safety_nnp_required_or_admin` — invariant.
  - `safety_filter_chain_bounded` — invariant.
  - `safety_filter_inherited_on_fork` — child.filters == parent.filters at fork.
  - `safety_filter_preserved_on_exec` — execve does not clear chain.
  - `safety_action_priority` — KILL_PROCESS < KILL_THREAD < TRAP < ERRNO < USER_NOTIF < TRACE < LOG < ALLOW.
  - `liveness_filter_runs` — every syscall on filtered task runs every filter once.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `set_mode_filter` post: chain.len += 1; no_new_privs == 1 | `Seccomp::set_mode_filter` |
| `set_mode_strict` post: mode == STRICT; no filters added | `Seccomp::set_mode_strict` |
| `run_filter_chain` post: returns dominant action | `Seccomp::run_filter_chain` |
| `tsync` post: on failure chain unchanged on all threads | `Seccomp::tsync` |

### Layer 4: Verus / Creusot functional

Per-`seccomp(2)` man-page semantic equivalence. LTP `seccomp01..seccomp10`, kselftest `seccomp_bpf` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`seccomp(2)` reinforcement:

- **Per-NNP or CAP_SYS_ADMIN gate** — defense against per-filter-bypass-via-uid-0-exec.
- **Per-NNP forced after FILTER install** — defense against per-filter-stripped-via-suid-exec.
- **Per-`BPF_MAXINSNS` cap** — defense against per-DoS-via-huge-filter.
- **Per-MAX_FILTERS_PER_TASK = 64** — defense against per-chain-overflow.
- **Per-action priority (KILL_PROCESS dominates)** — defense against per-permissive-filter-shadowing.
- **Per-`seccomp_data.arch` exposure** — defense against per-cross-arch-syscall confusion (if filter checks `arch`).
- **Per-LIFO evaluation** — every installed filter runs; cannot be skipped.
- **Per-USER_NOTIF blocking** — defense against per-syscall-race during supervisor decision.
- **Per-`TSYNC` atomic** — defense against per-thread-race window where one thread is filtered and another is not.

### grsecurity / pax surface

- **PaX UDEREF on `sock_fprog` and filter instruction copy** — defense against per-filter-pointer kernel-deref bug.
- **PAX_RANDKSTACK at seccomp entry** — randomizes kernel stack offset per call.
- **PaX KERNEXEC enforcement on cBPF→eBPF JIT** — JIT output mapped read-execute, never read-write-execute.
- **GRKERNSEC_HARDEN_PTRACE interaction** — ptrace-attached supervisor that responds to USER_NOTIF must satisfy grsec process-isolation policy.
- **GRKERNSEC: PR_SET_NO_NEW_PRIVS mandatory for sandboxes** — grsec policy refuses `SECCOMP_SET_MODE_FILTER` via `CAP_SYS_ADMIN` path; mandates NNP-first.
- **GRKERNSEC_CHROOT_CAPS** — chroot strips `CAP_SYS_ADMIN`, ensuring `seccomp` inside chroot requires NNP.
- **Per-grsec audit of every filter install** — instruction count, action distribution, install site PC logged.
- **Per-grsec seccomp-bypass detector** — runtime monitor flags any task that installs zero-instruction "allow-all" filters (sandbox-escape signal).
- **PaX MPROTECT integration** — seccomp `SPEC_ALLOW` flag (which skips SSBD) is refused by PaX policy; forces SSBD on.
- **PaX RAP** — seccomp filter JIT control-flow integrity checked against RAP function-type hashes.

