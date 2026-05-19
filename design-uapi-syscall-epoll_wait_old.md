---
title: "Tier-5 syscall: epoll_wait_old(2) — syscall 215"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`epoll_wait_old(2)` is a **deprecated x86_64 syscall slot** (number 215) reserved during the original epoll merge for an early prototype of `epoll_wait(2)`. Like `epoll_ctl_old` (214), the prototype was abandoned before any libc shipped a wrapper; the slot is permanently bound to `sys_ni_syscall` and unconditionally returns `-ENOSYS`.

The slot exists for x86_64 ABI stability — once a number is exposed in the syscall table, it cannot be reused without breaking ABI. So 215 is permanently dead alongside 214.

Critical for: ABI-stability historical record; security audit of `-ENOSYS` syscall slots; defense against malware probing for hidden kernel handlers.

### Acceptance Criteria

- [ ] AC-1: `syscall(__NR_epoll_wait_old, 0, 0, 0, 0)` returns -1 with errno=ENOSYS.
- [ ] AC-2: `syscall(__NR_epoll_wait_old, ...)` does not crash or hang regardless of arguments.
- [ ] AC-3: strace of a program calling 215 prints `epoll_wait_old(0, NULL, 0, 0) = -1 ENOSYS`.
- [ ] AC-4: No LSM hook fires for syscall 215.
- [ ] AC-5: No memory allocation, no copy_from_user, no permission check executes.
- [ ] AC-6: Tracing `sys_enter` for nr=215 fires once per call.
- [ ] AC-7: seccomp rule on __NR_epoll_wait_old behaves like seccomp on any other syscall.
- [ ] AC-8: The syscall table entry is `sys_ni_syscall`.

### Architecture

```rust
#[syscall(nr = 215, abi = "sysv", deprecated)]
pub fn sys_epoll_wait_old(_a: u64, _b: u64, _c: u64, _d: u64) -> isize {
    Errno::ENOSYS.into_isize()
}
```

Bound to the global `sys_ni_syscall` stub identical to syscall 214's binding:

`Stub::ni_syscall() -> isize`:
1. /* No argument access, no allocation. */
2. /* No LSM hook. */
3. /* Optional audit record. */
4. Errno::ENOSYS.into_isize()

The syscall dispatcher resolves 215 to `sys_ni_syscall` without any wrapper-level argument copy.

### Out of Scope

- `epoll_wait(2)` (separate Tier-5 doc — syscall 232, the live API).
- `epoll_pwait(2)` (separate Tier-5 doc — syscall 281, signal-mask variant).
- `epoll_pwait2(2)` (separate Tier-5 doc — modern timespec variant, syscall 441).
- `epoll_ctl_old(2)` (separate Tier-5 doc — syscall 214, sibling ENOSYS slot).
- `sys_ni_syscall` infrastructure (Tier-3 `kernel/sys_ni.md`).
- Implementation code.

### signature

```c
/* No libc header declares this. The historical intent was: */

int epoll_wait_old(int epfd, struct epoll_event *events, int maxevents, int timeout);

/* This prototype never shipped in any released kernel. */
```

### parameters

(unused — kernel returns ENOSYS before parameter parsing)

| Name | Type | Direction | Description |
|---|---|---|---|
| `epfd` | `int` | (unused) | (would have been epoll fd) |
| `events` | `struct epoll_event *` | (unused) | (would have been event output buffer) |
| `maxevents` | `int` | (unused) | (would have been max events to return) |
| `timeout` | `int` | (unused) | (would have been timeout in ms) |

### return value

| Value | Meaning |
|---|---|
| `-1` + `errno=ENOSYS` | Unconditional; this syscall is permanently unimplemented. |

### errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always. Kernel returns immediately from `sys_ni_syscall`. |

### abi surface

```text
__NR_epoll_wait_old  (x86_64)   = 215

/* Not allocated on i386, arm64, riscv64, or any other architecture.
   This is an x86_64-only ABI-reserved slot. */

/* Kernel binding: */
215  64  epoll_wait_old  sys_ni_syscall
```

### compatibility contract

REQ-1: Syscall number is **215** on x86_64 only. Not present on i386, arm64, riscv64, or any generic-syscall arch.

REQ-2: The kernel handler is unconditionally `sys_ni_syscall`, which returns `-ENOSYS` immediately. No argument is read, no permission check, no LSM hook.

REQ-3: The slot has never been implemented in any released Linux kernel. The prototype was supplanted by `epoll_wait(2)` (number 232) before merging.

REQ-4: The slot MUST remain `sys_ni_syscall` indefinitely; reusing it for a different syscall would silently change behavior for any program that probes the slot directly.

REQ-5: glibc does not provide a `syscall(__NR_epoll_wait_old, ...)` wrapper; programs invoking it must construct the syscall manually.

REQ-6: `strace` recognizes the syscall number and prints `epoll_wait_old(...)`; result is `-1 ENOSYS`.

REQ-7: Audit subsystem records syscall 215 and the ENOSYS return; no `AUDIT_PATH` or `AUDIT_FD` records emit.

REQ-8: seccomp filters can match `__NR_epoll_wait_old` as a sentinel; allowing or denying does not change kernel behavior.

REQ-9: kprobe / tracepoint on `sys_enter_epoll_wait_old` fires before the ENOSYS return; useful for probing-attempt monitoring.

REQ-10: Any future "revival" of this slot would constitute an ABI break and is policy-prohibited.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_returns_enosys` | INVARIANT | every invocation returns -ENOSYS. |
| `no_argument_read` | INVARIANT | no copy_from_user fires. |
| `no_allocation` | INVARIANT | no kmalloc / vmalloc. |
| `no_lsm_hook` | INVARIANT | no security_* hook fires. |
| `no_side_effect` | INVARIANT | task state, mm, fd table unchanged. |

### Layer 2: TLA+

`kernel/ni-syscall.tla` (shared with epoll_ctl_old, tuxcall, security):
- States: per-entry, per-return.
- Properties:
  - `safety_no_state_change` — no kernel state mutates.
  - `safety_constant_return` — always ENOSYS.
  - `liveness_immediate` — returns in O(1).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_epoll_wait_old` post: returns -ENOSYS | `Stub::ni_syscall` |
| `sys_epoll_wait_old` post: no allocation, no copy | `Stub::ni_syscall` |

### Layer 4: Verus / Creusot functional

Per-`sys_ni_syscall` semantics: returns ENOSYS, no side effects. ABI-stable since the epoll merge.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`epoll_wait_old(2)` reinforcement:

- **Per-ENOSYS-strict response** — defense against per-future-revival ABI break.
- **Per-no-argument-read** — defense against per-arg-fuzz DoS (no syscall-arg parsing path).
- **Per-no-allocation** — defense against per-ENOSYS-allocator-pressure attack.
- **Per-no-LSM-hook** — defense against per-policy-spoofing via never-implemented syscall.
- **Per-audit-attempt** — defense against per-stealth-probe by malware (any caller is loggable).

### grsecurity / pax surface

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every invocation of an ENOSYS-deprecated slot (epoll_ctl_old=214, epoll_wait_old=215, tuxcall=184, security=185, vserver=236, getpmsg=181, putpmsg=182, afs_syscall=183) is audited with caller PID, comm, and process tree. Defense against malware probing for hidden syscall implementations.
- **PaX KERNEXEC neutral** — no exec; bounded.
- **PaX UDEREF neutral** — no copy_from_user; bounded.
- **GRKERNSEC_HIDESYM** — `sys_ni_syscall` symbol not exposed via /proc/kallsyms.
- **PAX_USERCOPY_HARDEN neutral** — no usercopy.
- **GRKERNSEC_BRUTE_PROTECT** — repeated ENOSYS probes from the same task in a short window may trigger brute-force-protection signal escalation.
- **No CAP requirement** — ENOSYS is unprivileged; grsec preserves this but adds the audit trail.
- **No_new_privs neutral** — ENOSYS slot is not a privilege boundary.
- **Permanent ABI freeze** — grsec policy treats the slot as immutable.
- **seccomp interaction** — grsec respects seccomp filters on __NR_epoll_wait_old; audit trail is independent of seccomp.
- **No syscall-table mutation** — grsec verifies at boot that sys_call_table[215] == sys_ni_syscall; divergence triggers a panic in PAX_HARDENED.
- **GRKERNSEC_AUDIT_GROUP** — audit can be restricted to specific group memberships to avoid log flooding.
- **Co-located with epoll_ctl_old** — grsec audits sequential probing of 214 then 215 (a fingerprint of malware enumerating ENOSYS slots) more aggressively than isolated calls.

### historical context

The original epoll merge (2.5.45, Davide Libenzi, 2002) introduced three syscalls during prototyping:

- `epoll_create` (number 213): create epoll instance fd.
- `epoll_ctl_old` (number 214): early prototype of `epoll_ctl`.
- `epoll_wait_old` (number 215): early prototype of `epoll_wait`.

Before the final merge, the API was redesigned: `epoll_ctl` and `epoll_wait` (numbers 232, 233) were given the cleaner signatures that ship today. The prototypes at 214/215 were never wired to handlers; the slots have been `sys_ni_syscall` since the merge landed.

Other architectures (i386, arm64, riscv64) never inherited the prototype slots — they use only the modern `epoll_create=213/254`, `epoll_ctl=233/21`, `epoll_wait=232/22` numbers (varies per arch).

The ABI-stability commitment is that **x86_64 syscall 215 will always be ENOSYS**. Any future kernel that wires it to a real handler would silently change semantics for any caller of `syscall(215, ...)`. Modern epoll improvements (e.g. `epoll_pwait2(2)` syscall 441 with timespec timeout) get fresh syscall numbers, not the dead slots.

### forensic indicators

A process that issues `syscall(215, ...)` is almost certainly one of:

1. A buggy program that miscomputed `__NR_epoll_wait` (typo / stale header).
2. A static-analyzer or system-call enumerator (calibration probes).
3. Malware fingerprinting the kernel by probing ENOSYS slots to distinguish stock from custom kernels — particularly the 214→215 sequential probe pattern, which is a well-known recon signature.

Case 3 is the security-relevant pattern; grsec audit records (configured via GRKERNSEC_DEPRECATED_SYSCALL_AUDIT) make it observable, and the 214→215 sequential pattern is flagged with elevated severity because it has no benign interpretation outside of a static-analyzer running with --enumerate.

