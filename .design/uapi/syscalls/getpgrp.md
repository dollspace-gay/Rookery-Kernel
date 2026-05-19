# Tier-5 syscall: getpgrp(2) — syscall 111

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE0(getpgrp))
  - include/linux/sched/signal.h (task_pgrp, task_pgrp_vnr)
  - include/linux/pid.h (pid_vnr)
  - arch/x86/entry/syscalls/syscall_64.tbl (111  common  getpgrp)
-->

## Summary

`getpgrp(2)` returns the **process-group ID** (PGID) of the **calling** process, mapped into the caller's pid-namespace. Equivalent to `getpgid(0)`. Predates `getpgid(2)` in System V Unix; retained for BSD/POSIX compatibility and because it cannot fail (no argument to misuse).

POSIX defines `getpgrp()` as the no-argument form returning the caller's PGID; BSD originally had a one-argument form, which Linux exposes only as `getpgid(2)`. Modern code uses either `getpgrp()` (no-arg) or `getpgid(0)` interchangeably.

Critical for: shell job-control startup (`bash` checks if its PGID is the foreground group via `tcgetpgrp(STDIN) == getpgrp()`), libc cleanup paths, daemon self-introspection, audit subsystem's group-correlation, `nohup` to determine if detachment is needed.

## Signature

```c
pid_t getpgrp(void);
```

## Parameters

(none — `SYSCALL_DEFINE0(getpgrp)`)

## Return value

| Value | Meaning |
|---|---|
| `pid_t` | The caller's PGID mapped into `current_pid_ns()`. |

`getpgrp()` cannot fail. POSIX-mandated: never returns -1, never sets errno.

## Errors

(none)

## ABI surface

```text
__NR_getpgrp (x86_64)   = 111
__NR_getpgrp (i386)     = 65
__NR_getpgrp (arm64)    = N/A (generic does not have getpgrp — replaced by getpgid(0))

typedef __kernel_pid_t  pid_t;   /* signed int */

struct signal_struct {
    ...
    struct pid *pids[PIDTYPE_MAX];   /* PIDTYPE_PGID */
    ...
};
```

Note: on generic-syscall architectures (arm64, riscv, etc.), the syscall `getpgrp` is NOT defined; userspace libc emulates it via `getpgid(0)`. This Tier-5 doc covers the x86_64/i386 syscall path; the generic-arch path falls back to `getpgid`.

## Compatibility contract

REQ-1: Syscall number is **111** on x86_64; **65** on i386. NOT present on arm64 / generic — libc emulates via `getpgid(0)` syscall 155.

REQ-2: Returns `task_pgrp_vnr(current)` = `pid_nr_ns(current->signal->pids[PIDTYPE_PGID], task_active_pid_ns(current))`.

REQ-3: Equivalent in semantics to `getpgid(0)`. The two syscalls return identical values for the same caller.

REQ-4: All threads in a thread group share the same PGID; `getpgrp()` returns the thread group's PGID.

REQ-5: After fork: child inherits parent's PGID. After execve: PGID preserved.

REQ-6: After `setpgid(0, X)`: `getpgrp()` returns X. After `setsid()`: `getpgrp() == getpid()`.

REQ-7: `getpgrp()` is async-signal-safe and re-entrant; RCU-read of `current->signal->pids[PIDTYPE_PGID]`.

REQ-8: No LSM hook on the getpgrp path.

REQ-9: `getpgrp()` does NOT mutate any task state. Pure read.

REQ-10: Returns NEVER < 0 (pid_t is signed but PGIDs are positive). 0 is impossible (a process always has a PGID in its own active pid-ns; pid_nr_ns of own task always returns positive).

REQ-11: Inside a pid-namespace: returns PGID in that pid-ns. Cross-ns reach-up not possible (current always has a visible pid in its own ns).

REQ-12: POSIX BSD historical: some BSD systems had `getpgrp(pid)`; Linux does NOT accept arguments — the syscall is `SYSCALL_DEFINE0`. Userspace libc emulates BSD form via `getpgid(pid)`.

REQ-13: `getpgrp()` is one of the cheapest syscalls (single RCU-pid-translation; no locks, no permissions).

## Acceptance Criteria

- [ ] AC-1: Process in own group: `getpgrp() == getpid()` (e.g. session leader or after setpgid(0,0)).
- [ ] AC-2: Process in parent's group (newly forked, no setpgid): `getpgrp() == parent's pgid pre-fork`.
- [ ] AC-3: After `setsid()`: `getpgrp() == getpid()`.
- [ ] AC-4: After `setpgid(0, X)`: `getpgrp() == X`.
- [ ] AC-5: All threads of a process return same `getpgrp()`.
- [ ] AC-6: `getpgrp()` consistent with `getpgid(0)`.
- [ ] AC-7: `getpgrp()` consistent with `/proc/self/stat` pgrp field.
- [ ] AC-8: After execve: `getpgrp()` unchanged.
- [ ] AC-9: `getpgrp()` never returns 0 or negative.
- [ ] AC-10: Concurrent `setpgid()` from sibling: `getpgrp()` observes pre- or post-state atomically.
- [ ] AC-11: Inside pid-ns: `getpgrp()` returns PGID in that ns (not host-ns value).

## Architecture

```rust
#[syscall(nr = 111, abi = "sysv")]
pub fn sys_getpgrp() -> pid_t {
    Getpgrp::do_getpgrp()
}
```

`Getpgrp::do_getpgrp() -> pid_t`:
1. let task = current();
2. /* RCU-read PIDTYPE_PGID slot */
3. let pgid = rcu::read(|| {
4.     PidNs::pid_vnr(task.signal.pids[PIDTYPE_PGID])
5. });
6. /* Invariant: pgid > 0 for live task in own ns */
7. pgid

`task_pgrp_vnr(task) -> pid_t`:
1. pid_nr_ns(task.signal.pids[PIDTYPE_PGID], task_active_pid_ns(current()))

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pgid_positive` | INVARIANT | per-getpgrp: returns > 0 for live task. |
| `equivalent_to_getpgid_zero` | INVARIANT | per-getpgrp: ret == getpgid(0). |
| `rcu_atomic` | INVARIANT | per-getpgrp: concurrent setpgid yields consistent snapshot. |
| `no_state_mutation` | INVARIANT | per-getpgrp: read-only. |
| `pid_ns_scoped` | INVARIANT | per-getpgrp: returned value is in caller's active pid-ns. |

### Layer 2: TLA+

`kernel/getpgrp.tla`:
- States: per-task PGID, pid-ns.
- Properties:
  - `safety_positive_return` — getpgrp always returns positive PGID.
  - `safety_equiv_getpgid_zero` — getpgrp() == getpgid(0) at all times.
  - `safety_no_mutation` — getpgrp never mutates state.
  - `liveness_returns` — getpgrp terminates in O(1).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getpgrp` post: ret == task_pgrp_vnr(current) | `Getpgrp::do_getpgrp` |
| `do_getpgrp` post: ret > 0 | `Getpgrp::do_getpgrp` |
| `do_getpgrp` post: equivalent to do_getpgid(0) | `Getpgrp::do_getpgrp` |

### Layer 4: Verus / Creusot functional

Per-`getpgrp(2)` man-page equivalence. LTP `getpgrp01`, `getpgrp02` pass. POSIX.1-2008 `getpgrp` semantics verified.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getpgrp(2)` reinforcement:

- **Per-RCU snapshot** — defense against per-torn-pid-read race with concurrent setpgid.
- **Per-pid-ns-translation** — defense against per-namespace-leak (kernel-global group leader pid would otherwise leak).
- **Per-readonly semantics** — defense against per-mutation-via-getpgrp bug pattern.
- **Per-no-argument** — defense against per-untrusted-pid-input (cannot probe for hidden tasks).

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at getpgrp entry** — randomizes kernel stack offset; defeats stack-layout fingerprinting via high-frequency `getpgrp()` probes (note: getpgrp is a hot syscall in shells and libcs).
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/self/stat`'s pgrp field is restricted to owning user and CAP_SYS_ADMIN; `getpgrp()` returns own PGID with no info-leak concern (self-data only).
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a grsec chroot, `getpgrp()` returns PGID in chroot's pid-ns; never the host-ns PGID.
- **CAP_SETUID / CAP_SETGID strict** — getpgrp is a pure read; not gated by cred-change caps.
- **GRKERNSEC_AUDIT_GROUP on cred-change** — getpgrp is not itself a cred-change; logged in correlation with setpgid for audited users.
- **GRKERNSEC_HIDESYM** — getpgrp returns only the PGID; no kernel pointers exposed.
- **No LSM hook** — getpgrp has no security hook; grsec adds no checks (read-only, no-argument).
- **No info-leak surface** — getpgrp has no input from userspace; cannot be used to probe for hidden tasks.
- **PAX_USERCOPY honored** — getpgrp returns scalar; no user buffer; safe.
- **Stack canary honored** — kernel-stack canary protects the syscall entry frame.
- **Cheap-syscall optimization** — grsec does not add audit overhead per-call unless explicitly enabled, since getpgrp is a hot path in shells.
- **Rate-limit-friendly** — when grsec audit IS enabled, getpgrp calls are aggregated/coalesced (e.g. one audit record per N calls) to avoid log-flood denial-of-service.
- **PGID zombie correctness** — if the group leader is a zombie (exited but not reaped), getpgrp continues to return the (zombie) leader's pid in caller's ns; grsec validates that the pid is still allocated (not yet recycled) before returning.
- **Cross-architecture parity** — grsec hardening of getpgrp on x86_64/i386 is equivalent to the hardening of getpgid(0) on arm64/generic; no security regression across architectures.
- **No probe surface** — getpgrp has zero user-controlled input and a single output. Grsec confirms there is no observable side-channel (timing, errno, kernel-state) by which an attacker could leak information beyond the caller's own PGID — which the caller is already entitled to know.
- **Container init friendliness** — container init processes (PID 1 in a pid-ns) call getpgrp during early bootstrap; grsec's chroot-findtask scoping ensures the returned PGID is in the container's pid-ns, supporting container introspection without leaking host-ns identity.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `getpgid(2)` (Tier-5 separate doc — accepts a pid argument).
- `setpgid(2)` (Tier-5 separate doc).
- `getsid(2)` / `setsid(2)` (Tier-5 separate docs).
- BSD-style `getpgrp(pid)` (libc-emulated via `getpgid`).
- Implementation code.
