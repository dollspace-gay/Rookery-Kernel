---
title: "Tier-5 syscall: epoll_ctl_old(2) — syscall 214"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`epoll_ctl_old(2)` is a **deprecated x86_64 syscall slot** (number 214) reserved for a never-released early prototype of `epoll_ctl(2)`. The slot was allocated during the original epoll merge (2.5/2.6 era) but the prototype was supplanted by `epoll_ctl(2)` (number 233) before any libc shipped a wrapper. The syscall has **never been implemented** in any released kernel: every entry in the syscall table is `sys_ni_syscall`, which returns `-ENOSYS`.

The slot exists for ABI stability — once a syscall number is exposed in the syscall table on x86_64, it cannot be reused for an unrelated call without breaking ABI promises. So 214 is permanently dead.

Critical for: ABI-stability historical record; security audit of `-ENOSYS` syscall slots; defense against malware probing for hidden kernel handlers.

### Acceptance Criteria

- [ ] AC-1: `syscall(__NR_epoll_ctl_old, 0, 0, 0, 0)` returns -1 with errno=ENOSYS.
- [ ] AC-2: `syscall(__NR_epoll_ctl_old, ...)` does not crash or hang regardless of argument values.
- [ ] AC-3: `strace` of a program calling 214 prints `epoll_ctl_old(0, 0, 0, NULL) = -1 ENOSYS`.
- [ ] AC-4: No LSM hook fires for syscall 214.
- [ ] AC-5: No memory allocation, no copy_from_user, no permission check executes.
- [ ] AC-6: Tracing `sys_enter` for nr=214 fires once per call.
- [ ] AC-7: seccomp rule on __NR_epoll_ctl_old behaves like seccomp on any other syscall (filter executes; action applies).
- [ ] AC-8: The syscall table entry is `sys_ni_syscall` (verifiable via /proc/kallsyms or ftrace).

### Architecture

```rust
#[syscall(nr = 214, abi = "sysv", deprecated)]
pub fn sys_epoll_ctl_old(_a: u64, _b: u64, _c: u64, _d: u64) -> isize {
    Errno::ENOSYS.into_isize()
}
```

`sys_epoll_ctl_old` is bound to the global `sys_ni_syscall` stub which is:

`Stub::ni_syscall() -> isize`:
1. /* No argument access, no allocation. */
2. /* No LSM hook (this is intentional — ENOSYS is the canonical "syscall not present" answer). */
3. /* Optional: audit subsystem records the attempt if the task is in an audit group. */
4. Errno::ENOSYS.into_isize()

The syscall-entry assembly path looks the call up in the per-arch syscall table; for x86_64 entry 214 resolves to `sys_ni_syscall` and the dispatcher invokes it without any wrapper-level argument copy.

### Out of Scope

- `epoll_ctl(2)` (separate Tier-5 doc — syscall 233, the live API).
- `epoll_wait_old(2)` (separate Tier-5 doc — syscall 215, also ENOSYS).
- `epoll_pwait2(2)` (separate Tier-5 doc — modern timespec variant).
- `sys_ni_syscall` infrastructure (Tier-3 `kernel/sys_ni.md`).
- ABI-freeze policy (project policy doc).
- Implementation code.

### signature

```c
/* No libc header declares this. The historical signature was intended to be: */

int epoll_ctl_old(int epfd, int op, int fd, struct epoll_event *event);

/* But this prototype never reached any release kernel. */
```

### parameters

(unused — kernel returns ENOSYS before parameter parsing)

| Name | Type | Direction | Description |
|---|---|---|---|
| `epfd` | `int` | (unused) | (would have been epoll fd) |
| `op` | `int` | (unused) | (would have been EPOLL_CTL_ADD/MOD/DEL) |
| `fd` | `int` | (unused) | (would have been target fd) |
| `event` | `struct epoll_event *` | (unused) | (would have been event spec) |

### return value

| Value | Meaning |
|---|---|
| `-1` + `errno=ENOSYS` | Unconditional; this syscall is permanently unimplemented. |

### errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always. The kernel returns immediately from `sys_ni_syscall`. |

### abi surface

```text
__NR_epoll_ctl_old  (x86_64)   = 214

/* Not allocated on i386, arm64, riscv64, or any other architecture.
   This is an x86_64-only ABI-reserved slot. */

/* Kernel binding: */
214  64  epoll_ctl_old  sys_ni_syscall
```

### compatibility contract

REQ-1: Syscall number is **214** on x86_64 only. Not present on i386, arm64, riscv64, or any generic-syscall arch.

REQ-2: The kernel handler is unconditionally `sys_ni_syscall`, which returns `-ENOSYS` immediately. No argument is read, no permission check fires, no LSM hook fires.

REQ-3: The slot has never been implemented in any released Linux kernel. The historical commit log shows the slot was reserved during the epoll merge and the prototype was abandoned before any libc wrapper shipped.

REQ-4: The slot MUST remain `sys_ni_syscall` indefinitely; reusing 214 for a different syscall would silently change behavior for any program that probes the slot via the syscall ABI directly.

REQ-5: glibc does not provide a `syscall(__NR_epoll_ctl_old, ...)` wrapper; programs invoking it must construct the syscall instruction manually (or via `syscall(2)` numeric).

REQ-6: `strace` recognizes the syscall number and prints `epoll_ctl_old(...)` for legibility, but the result is always `-1 ENOSYS`.

REQ-7: Audit subsystem records the syscall number (214) and the ENOSYS return; no `AUDIT_PATH` or `AUDIT_FD` records emit (no arguments are parsed).

REQ-8: seccomp filters can match `__NR_epoll_ctl_old` and may use it as a sentinel; allowing or denying does not change kernel behavior (the call always returns ENOSYS).

REQ-9: kprobe / tracepoint on `sys_enter_epoll_ctl_old` fires before the ENOSYS return; useful for forensic monitoring of probes.

REQ-10: Any future "revival" of this slot would constitute an ABI break and is policy-prohibited.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_returns_enosys` | INVARIANT | every invocation returns -ENOSYS. |
| `no_argument_read` | INVARIANT | no copy_from_user fires on syscall 214. |
| `no_allocation` | INVARIANT | no kmalloc / vmalloc on syscall 214. |
| `no_lsm_hook` | INVARIANT | no security_* hook fires. |
| `no_side_effect` | INVARIANT | task state, mm, fd table unchanged. |

### Layer 2: TLA+

`kernel/ni-syscall.tla`:
- States: per-entry, per-return.
- Properties:
  - `safety_no_state_change` — no kernel state mutates.
  - `safety_constant_return` — always ENOSYS.
  - `liveness_immediate` — returns in O(1).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_epoll_ctl_old` post: returns -ENOSYS | `Stub::ni_syscall` |
| `sys_epoll_ctl_old` post: no allocation, no copy | `Stub::ni_syscall` |

### Layer 4: Verus / Creusot functional

Per-`sys_ni_syscall` semantics: returns ENOSYS, no side effects. ABI-stable since the epoll merge.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`epoll_ctl_old(2)` reinforcement:

- **Per-ENOSYS-strict response** — defense against per-future-revival ABI break.
- **Per-no-argument-read** — defense against per-arg-fuzz DoS (no syscall-arg parsing path).
- **Per-no-allocation** — defense against per-ENOSYS-allocator-pressure attack.
- **Per-no-LSM-hook** — defense against per-policy-spoofing via never-implemented syscall.
- **Per-audit-attempt** — defense against per-stealth-probe by malware (any caller is loggable).

### grsecurity / pax surface

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every invocation of an ENOSYS-deprecated slot (epoll_ctl_old, epoll_wait_old, tuxcall, security, vserver, getpmsg, putpmsg, afs_syscall, set_thread_area legacy on x86_64) is audited with the caller PID, comm, and process tree. Defense against malware probing for hidden syscall implementations or "secret" backdoors.
- **PaX KERNEXEC neutral** — no exec; bounded.
- **PaX UDEREF neutral** — no copy_from_user; bounded.
- **GRKERNSEC_HIDESYM** — `sys_ni_syscall` symbol not exposed via /proc/kallsyms (the dispatcher resolves it but userspace cannot enumerate it directly).
- **PAX_USERCOPY_HARDEN neutral** — no usercopy.
- **GRKERNSEC_BRUTE_PROTECT** — repeated ENOSYS probes from the same task within a short window may trigger brute-force-protection (signal escalation) for tasks marked as such.
- **No CAP requirement** — ENOSYS is unprivileged; grsec preserves this but adds the audit trail.
- **No_new_privs neutral** — ENOSYS slot is not a privilege boundary.
- **Permanent ABI freeze** — grsec policy treats the ENOSYS slot as immutable: any kernel revival of 214 in a custom build is policy-flagged.
- **seccomp interaction** — grsec respects seccomp rules on __NR_epoll_ctl_old (caller can match and block); the audit trail is independent of seccomp.
- **No syscall-table mutation** — grsec verifies (at boot) that sys_call_table[214] == sys_ni_syscall; any divergence triggers a panic in PAX_HARDENED config.
- **GRKERNSEC_AUDIT_GROUP** — the audit can be restricted to specific group memberships to avoid log flooding.

### historical context

The original epoll merge (2.5.45, Davide Libenzi, 2002) introduced three syscalls during prototyping:

- `epoll_create` (number 213): create epoll instance fd.
- `epoll_ctl_old` (number 214): early prototype of `epoll_ctl`.
- `epoll_wait_old` (number 215): early prototype of `epoll_wait`.

Before the final merge, the API was redesigned: `epoll_ctl` and `epoll_wait` (numbers 232, 233) were given the cleaner signatures that ship today. The prototypes at 214/215 were never wired to handlers; the slots have been `sys_ni_syscall` since the merge landed.

Other architectures (i386, arm64, riscv64) never inherited the prototype slots — they use only the modern `epoll_create=213/254`, `epoll_ctl=233/21`, `epoll_wait=232/22` numbers (varies per arch).

The ABI-stability commitment is that **x86_64 syscall 214 will always be ENOSYS**. Any future kernel that wires it to a real handler — even hypothetically a "modernized epoll_ctl" — would silently change semantics for any caller of `syscall(214, ...)`, which would be a regression. Modern epoll improvements (e.g. `epoll_pwait2(2)` syscall 441 with timespec timeout) get fresh syscall numbers, not the dead slots.

### forensic indicators

A process that issues `syscall(214, ...)` is almost certainly one of:

1. A buggy program that miscomputed `__NR_epoll_ctl` (typo/copy-paste from a stale header).
2. A static-analyzer or system-call enumerator (e.g. `strace -y -c` with `--syscall-times` calibration probes).
3. Malware fingerprinting the kernel by probing ENOSYS slots to distinguish stock from custom kernels.

Case 3 is the security-relevant pattern; grsec audit records (configured via GRKERNSEC_DEPRECATED_SYSCALL_AUDIT) make it observable.

