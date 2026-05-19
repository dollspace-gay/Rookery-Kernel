# Tier-5 syscall: prlimit64(2) — syscall 302

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE4(prlimit64), do_prlimit)
  - include/uapi/asm-generic/resource.h (RLIMIT_*, RLIM_INFINITY)
  - include/uapi/linux/resource.h (struct rlimit64)
  - arch/x86/entry/syscalls/syscall_64.tbl (302  common  prlimit64)
-->

## Summary

`prlimit64(2)` reads and/or writes a resource limit on an arbitrary process. It is the modern, 64-bit, atomic-read-modify replacement for the legacy `getrlimit`/`setrlimit` pair, with three improvements: (a) operates on any pid (not just self), (b) uses `__u64` values everywhere (avoids the 32-bit `RLIM_INFINITY` ambiguity), (c) reads and writes in one syscall (`old_limit` returned alongside the `new_limit` being set). Libc's `getrlimit`/`setrlimit` since glibc 2.13 are thin wrappers over `prlimit64`.

Critical for: `ulimit`, systemd `LimitNOFILE=`, container runtimes setting RLIMIT_CORE=0, OOM-tuners adjusting RLIMIT_MEMLOCK, every CI runner enforcing CPU/AS/STACK limits.

## Signature

```c
int prlimit64(pid_t pid,
              int resource,
              const struct rlimit64 *new_limit,
              struct rlimit64 *old_limit);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pid` | `pid_t` | in | Target TGID (process). `0` = calling process. |
| `resource` | `int` | in | One of `RLIMIT_*` (0..15). |
| `new_limit` | `const struct rlimit64 *` | in | If non-NULL, the new `{rlim_cur, rlim_max}` to install. |
| `old_limit` | `struct rlimit64 *` | out | If non-NULL, receives the *previous* `{rlim_cur, rlim_max}`. |

```c
struct rlimit64 {
    __u64 rlim_cur;     /* soft limit */
    __u64 rlim_max;     /* hard limit (ceiling) */
};
```

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. If `new_limit != NULL`, the limit is updated; if `old_limit != NULL`, the prior values are returned. |
| `-1` + `errno` | Failure. No state mutation. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `new_limit` / `old_limit` user pointer invalid. |
| `EINVAL` | `resource` not in `[0, RLIM_NLIMITS)`; `new_limit->rlim_cur > new_limit->rlim_max`. |
| `EPERM`  | Raising hard limit without `CAP_SYS_RESOURCE`; cross-process call without `CAP_SYS_RESOURCE` or uid/gid match. |
| `ESRCH`  | `pid > 0` but no such task. |

## ABI surface

```text
__NR_prlimit64 (x86_64)  = 302
__NR_prlimit64 (i386)    = 340
__NR_prlimit64 (generic) = 261     /* arm64, riscv */

RLIMIT_* (asm-generic; most archs share these numbers)
   0  RLIMIT_CPU           /* CPU seconds (soft = SIGXCPU once, then SIGKILL; hard = unconditional kill) */
   1  RLIMIT_FSIZE         /* Max file size in bytes; soft excess -> SIGXFSZ */
   2  RLIMIT_DATA          /* Max data segment size (brk + initialized data) */
   3  RLIMIT_STACK         /* Max stack vma size in bytes */
   4  RLIMIT_CORE          /* Max coredump file size; 0 disables coredump */
   5  RLIMIT_RSS           /* Max resident set (advisory only on Linux; not enforced) */
   6  RLIMIT_NPROC         /* Max number of tasks for this real uid */
   7  RLIMIT_NOFILE        /* One past max fd number */
   8  RLIMIT_MEMLOCK       /* Max mlocked bytes */
   9  RLIMIT_AS            /* Max address-space size (sum of all vmas) */
  10  RLIMIT_LOCKS         /* Max number of fcntl(F_SETLK) locks (historically) */
  11  RLIMIT_SIGPENDING    /* Max queued signals (per-uid) */
  12  RLIMIT_MSGQUEUE      /* Max POSIX message-queue bytes per uid */
  13  RLIMIT_NICE          /* Soft nice ceiling: 20 - rlim_cur (negative-going) */
  14  RLIMIT_RTPRIO        /* Max SCHED_FIFO/RR rt-priority */
  15  RLIMIT_RTTIME        /* Max microseconds a SCHED_RR/FIFO task can run continuously */
  RLIM_NLIMITS = 16

Sentinels
  RLIM_INFINITY  = ~0ULL = 0xFFFFFFFFFFFFFFFF

Compat ABI (32-bit)
  prlimit64 always uses 64-bit struct, even on i386. struct rlimit (legacy) is 32-bit per field.
```

## Compatibility contract

REQ-1: Syscall number is **302** on x86_64; **261** on generic-syscall archs. ABI-stable.

REQ-2: `pid == 0`: target is the calling process (TGID).

REQ-3: `pid > 0`: target is that TGID. Caller must satisfy the cross-process check:
  - effective uid + gid match target's real or saved uid + gid, AND
  - effective uid + gid match target's real or saved uid + gid,
  - OR caller has `CAP_SYS_RESOURCE` in the target's user-namespace.

REQ-4: `resource` MUST be in `[0, RLIM_NLIMITS)` = `[0, 16)`. Else `-EINVAL`. New resources append to the end.

REQ-5: `new_limit` and `old_limit` MAY each independently be NULL. Both NULL → no-op + return 0 (effectively "validate pid+resource only"). NEW behavior was permitted from kernel ≥ 2.6.36.

REQ-6: If `new_limit != NULL`:
  - `new_limit->rlim_cur <= new_limit->rlim_max` (soft ≤ hard). Else `-EINVAL`.
  - Raising `rlim_max` requires `CAP_SYS_RESOURCE` in target's user-namespace.
  - Lowering `rlim_max` is permitted unconditionally (hard limit is irreversible-shrinking per-process).
  - Soft limit may move freely within `[0, rlim_max]`.

REQ-7: If `old_limit != NULL`: kernel writes the *pre-update* (or current, if no update) values. The read+write is atomic — concurrent prlimit64 calls cannot interleave to produce a torn snapshot.

REQ-8: Per-resource enforcement points are scattered (not in prlimit64 itself):
  - `RLIMIT_NOFILE`: enforced in `__alloc_fd`.
  - `RLIMIT_NPROC`: enforced in `copy_process` (fork).
  - `RLIMIT_AS`: enforced in `mmap_region`.
  - `RLIMIT_STACK`: enforced in stack-vma expansion.
  - `RLIMIT_FSIZE`: enforced in `write` paths.
  - `RLIMIT_CORE`: enforced in `do_coredump`.
  - `RLIMIT_CPU`: enforced in scheduler tick.
  - `RLIMIT_MEMLOCK`: enforced in `mlock` family.
  - `RLIMIT_MSGQUEUE`: enforced in `mq_open` and message enqueue.
  - `RLIMIT_NICE` / `RLIMIT_RTPRIO`: enforced in `setpriority`/`sched_setscheduler`.

REQ-9: `RLIMIT_NOFILE` hard limit: kernel-wide ceiling = `sysctl_nr_open` (default 1024*1024). Raising hard above this → `-EPERM` even with CAP_SYS_RESOURCE.

REQ-10: `RLIMIT_CORE = 0`: coredump suppressed completely. Even `kernel.core_pattern` piped-handlers do not run.

REQ-11: `RLIMIT_RTPRIO` interaction with `SCHED_FIFO`/`SCHED_RR`: if `rlim_max = 0`, the task cannot ask for any rt-priority unless it has `CAP_SYS_NICE`.

REQ-12: `RLIMIT_RTTIME`: when exceeded, `SCHED_FIFO/RR` task receives `SIGXCPU` at soft, `SIGKILL` at hard.

REQ-13: `RLIMIT_STACK` set < currently-used stack size: future fork inherits, but current task's existing stack vma is NOT shrunk in place; subsequent stack growth fails.

REQ-14: All updates are RCU-visible; observers see either pre or post.

REQ-15: Audit: AUDIT_RLIMIT record emitted on update (with old + new values).

REQ-16: Concurrent setrlimit/getrlimit/prlimit64 are serialized per-task via `tasklist_lock` + per-rlimit `seqlock`.

REQ-17: A child inherits all limits from its parent at fork; execve does NOT reset.

REQ-18: `rlim_cur > rlim_max` on existing state (e.g. via legacy 32-bit setrlimit clamping bug) is sanitized: kernel clamps to `rlim_max` when reading.

## Acceptance Criteria

- [ ] AC-1: `prlimit64(0, RLIMIT_NOFILE, NULL, &old)` reports current values.
- [ ] AC-2: `prlimit64(0, RLIMIT_NOFILE, &{4096, 4096}, NULL)` lowers limit to 4096.
- [ ] AC-3: `prlimit64(0, RLIMIT_NOFILE, &{2048, 8192}, NULL)` (raising hard) without `CAP_SYS_RESOURCE` returns `-EPERM`.
- [ ] AC-4: With `CAP_SYS_RESOURCE`, raise hard up to `sysctl_nr_open` succeeds.
- [ ] AC-5: Raise hard above `sysctl_nr_open` returns `-EPERM`.
- [ ] AC-6: `prlimit64(0, RLIMIT_NOFILE, &{cur, max}, &old)` returns old values AND installs new.
- [ ] AC-7: `prlimit64(0, RLIMIT_NOFILE, &{cur > max, max}, NULL)` returns `-EINVAL`.
- [ ] AC-8: `prlimit64(0, 16, NULL, NULL)` (RLIM_NLIMITS) returns `-EINVAL`.
- [ ] AC-9: `prlimit64(other_uid_pid, RLIMIT_NOFILE, NULL, &old)` without `CAP_SYS_RESOURCE` returns `-EPERM`.
- [ ] AC-10: `prlimit64(0, RLIMIT_CORE, &{0, 0}, NULL)` then segfault produces no coredump.
- [ ] AC-11: `prlimit64(getpid(), RLIMIT_NPROC, &{0, 0}, NULL)` then `fork` returns `-EAGAIN`.
- [ ] AC-12: Two concurrent `prlimit64` calls on same task interleave atomically (no torn rlim_cur/rlim_max read).
- [ ] AC-13: `prlimit64(0, RLIMIT_NOFILE, NULL, NULL)` returns 0 (validation no-op).
- [ ] AC-14: `prlimit64(0, RLIMIT_NOFILE, NULL, NULL_PTR_BUT_NONZERO_FAULT)` returns `-EFAULT`.
- [ ] AC-15: AUDIT_RLIMIT record emitted on every successful set.
- [ ] AC-16: Limits inherited across `fork`.
- [ ] AC-17: Limits preserved across `execve`.

## Architecture

```rust
#[syscall(nr = 302, abi = "sysv")]
pub fn sys_prlimit64(
    pid:       i32,
    resource:  i32,
    new_limit: UserPtr<Rlimit64>,
    old_limit: UserPtr<Rlimit64>,
) -> isize {
    Prlimit::do_prlimit(pid, resource, new_limit, old_limit)
}
```

`Prlimit::do_prlimit(pid, resource, new_p, old_p) -> isize`:
1. if resource as u32 >= RLIM_NLIMITS { return Err(EINVAL); }
2. let target = match pid {
3.   0          => current().group_leader(),
4.   p if p > 0 => Task::find_by_tgid(p).ok_or(ESRCH)?,
5.   _          => return Err(EINVAL),
6. };
7. /* Cross-process cred check */
8. if !same_task(target, current()) {
9.   if !uid_gid_match(target) && !ns_capable(target.user_ns, CAP_SYS_RESOURCE) {
10.    return Err(EPERM);
11.  }
12. }
13. /* Copy in new_limit if requested */
14. let new = if !new_p.is_null() { Some(new_p.copy_in()?) } else { None };
15. /* Lock target's rlimit array, read old, validate, commit new */
16. let old = {
17.   let guard = target.lock_rlimits();
18.   let old = guard.rlim[resource as usize].clone();
19.   if let Some(n) = &new {
20.     if n.rlim_cur > n.rlim_max { return Err(EINVAL); }
21.     if n.rlim_max > old.rlim_max && !ns_capable(target.user_ns, CAP_SYS_RESOURCE) {
22.       return Err(EPERM);
23.     }
24.     if resource == RLIMIT_NOFILE as i32 && n.rlim_max > sysctl_nr_open() {
25.       return Err(EPERM);
26.     }
27.     /* LSM hook */
28.     security::task_setrlimit(target, resource, n)?;
29.     guard.rlim[resource as usize] = n.clone();
30.   }
31.   old
32. };
33. /* Copy out */
34. if !old_p.is_null() { old_p.copy_out(&old)?; }
35. /* Audit */
36. if new.is_some() { audit::log_rlimit(target, resource, &new.unwrap(), &old); }
37. Ok(0)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `resource_range` | INVARIANT | resource ∈ [0, RLIM_NLIMITS). |
| `soft_le_hard` | INVARIANT | post: rlim_cur ≤ rlim_max. |
| `hard_raise_requires_cap` | INVARIANT | new.rlim_max > old.rlim_max ⟹ CAP_SYS_RESOURCE. |
| `nofile_cap_bounded` | INVARIANT | RLIMIT_NOFILE.rlim_max ≤ sysctl_nr_open. |
| `cross_proc_cred_check` | INVARIANT | pid != self ⟹ uid/gid match ∨ CAP_SYS_RESOURCE. |
| `atomic_read_modify_write` | INVARIANT | old captures pre-state under same lock as commit. |

### Layer 2: TLA+

`kernel/prlimit.tla`:
- States: target.rlim[0..15] = (cur, max).
- Properties:
  - `safety_soft_le_hard` — invariant for every resource.
  - `safety_hard_monotone_without_cap` — without CAP_SYS_RESOURCE, hard never raised.
  - `safety_atomic_read_set` — old returned and new installed under single lock.
  - `safety_cross_uid_blocked` — cross-uid calls without cap denied.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_prlimit` post: target.rlim[r] == new ∨ unchanged on error | `Prlimit::do_prlimit` |
| `do_prlimit` post: old_p (if non-null) contains pre-state | `Prlimit::do_prlimit` |
| `validate_new(new, old)` post: returns Ok iff capabilities7 + sysctl rules satisfied | `Prlimit::validate_new` |

### Layer 4: Verus / Creusot functional

Per-`prlimit(2)` man-page semantic equivalence. LTP `prlimit01..prlimit04` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`prlimit64(2)` reinforcement:

- **Per-resource range check** — defense against per-out-of-bounds rlim_array write.
- **Per-soft ≤ hard invariant** — defense against per-malformed-limit confusion in enforcement sites.
- **Per-CAP_SYS_RESOURCE gate on hard-raise** — defense against per-self-elevation of limits.
- **Per-sysctl_nr_open ceiling on NOFILE** — defense against per-fd-table-exhaustion DoS.
- **Per-cross-uid cred check** — defense against per-cross-tenant limit injection.
- **Per-atomic-read-set** — defense against per-torn-snapshot in ulimit tooling.
- **Per-AUDIT_RLIMIT** — defense against per-limit-drop log elision.

## Grsecurity / PaX surface

- **PaX UDEREF on `new_limit`/`old_limit` user pointers** — defense against per-rlim-buffer kernel-deref bug.
- **PAX_RANDKSTACK at prlimit64 entry** — randomizes kernel stack offset.
- **GRKERNSEC_CHROOT_CAPS** — `CAP_SYS_RESOURCE` stripped on chroot entry; chroot'd tasks cannot raise their own hard limits.
- **GRKERNSEC_RESLOG** — every successful set of RLIMIT_NPROC, RLIMIT_NOFILE, RLIMIT_MEMLOCK, RLIMIT_AS, RLIMIT_CORE logged to the grsec audit trail.
- **Per-grsec RLIMIT_NPROC enforcement at runtime** — grsec enforces RLIMIT_NPROC per real-uid at all fork/clone callsites with stricter accounting than upstream.
- **Per-grsec RLIMIT_NOFILE max-cap** — grsec policy can pin a global ceiling lower than `sysctl_nr_open`.
- **Per-grsec cross-uid policy** — grsec disallows `CAP_SYS_RESOURCE` cross-uid pivots unless RBAC subject permits.
- **PaX MPROTECT** — RLIMIT_STACK lowering interacts with MPROTECT to keep stack vma non-executable even after `prlimit64`.
- **GRKERNSEC_BRUTE** — repeated `-EPERM` on hard-raise triggers per-uid rate-limit (privilege-probing detector).

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `setrlimit(2)`, `getrlimit(2)` (Tier-5 separate docs; thin wrappers around prlimit64).
- Per-resource enforcement sites (Tier-3 in respective subsystem docs).
- `sysctl_nr_open` semantics (Tier-5 in `uapi/sysctl/nr_open.md`).
- POSIX message queue accounting (Tier-3 in `ipc/mqueue.md`).
- Implementation code.
