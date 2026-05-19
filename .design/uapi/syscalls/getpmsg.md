# Tier-5 syscall: getpmsg(2) — syscall 181 (unimplemented stub)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys_ni.c (SYSCALL_DEFINE0(ni_syscall) — unimplemented stub)
  - arch/x86/entry/syscalls/syscall_64.tbl (181  common  getpmsg)
-->

## Summary

`getpmsg(2)` is a **reserved, unimplemented** syscall number in Linux. It is the SVR4 / STREAMS-era "get next priority message" primitive (`getpmsg`/`putpmsg`/`putmsg`/`getmsg` were the original STREAMS message-queue I/O API). Linux has never implemented STREAMS in mainline; the syscall slot is reserved so the numbering matches other Unix variants and so the LSB ABI table stays aligned. Calling it always returns `-ENOSYS`. Documented in Tier-5 for ABI completeness and to specify the strict ENOSYS contract that hardened builds and seccomp profiles must preserve. Critical for: ABI-table stability, audit-trail of probe-for-STREAMS attempts, hardened ENOSYS contract.

## Signature

```c
/* The historical SVR4 STREAMS signature (NOT implemented in Linux): */
int getpmsg(int fd, struct strbuf *ctlptr, struct strbuf *dataptr,
            int *bandp, int *flagsp);
```

```c
/* Reference: SVR4 struct strbuf. Never used by the Linux kernel. */
struct strbuf {
    int   maxlen;
    int   len;
    char *buf;
};
```

## Parameters

(Linux: ALL arguments ignored; syscall returns ENOSYS immediately.)

For historical/SVR4 reference only:

| Name | Type | Description |
|---|---|---|
| `fd` | `int` | STREAMS file descriptor. |
| `ctlptr` | `struct strbuf *` | Control-message buffer. |
| `dataptr` | `struct strbuf *` | Data-message buffer. |
| `bandp` | `int *` | Priority band of received message. |
| `flagsp` | `int *` | Flags (`MSG_HIPRI`, `MSG_BAND`, `MSG_ANY`). |

## Return value

| Value | Meaning |
|---|---|
| `-1` + `errno=ENOSYS` | Always (Linux mainline never implemented STREAMS). |

## Errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always — syscall unimplemented. |

## ABI surface

```text
__NR_getpmsg (x86_64) = 181
__NR_getpmsg (arm64)  = N/A
__NR_getpmsg (riscv)  = N/A
__NR_getpmsg (i386)   = 188

/* The syscall slot is RESERVED. The kernel's sys_call_table entry points
   at sys_ni_syscall, which returns -ENOSYS. */
/* glibc historically exposed no public wrapper; some BSD-compat layers
   exposed getpmsg() as ENOSYS at the libc level. */
```

## Compatibility contract

REQ-1: Syscall number is **181** on x86_64. ABI-reserved (number SHALL NOT be reassigned). Body returns `-ENOSYS`.

REQ-2: Linux MUST NOT implement STREAMS. This is policy: STREAMS would conflict with the BSD-socket model and pose a security review burden. The reservation prevents accidental reassignment.

REQ-3: Per-`sys_ni_syscall` dispatch: the syscall table entry routes to the generic "not implemented" stub which returns `-ENOSYS` without consulting arguments.

REQ-4: Per-arch: arm64, riscv, loongarch do NOT reserve this number — calls to it from a 32-bit personality syscall vector return `-ENOSYS` via the generic catch-all.

REQ-5: Per-audit: every call audit-logged with the syscall number and caller real-uid (defense against legacy-syscall probing).

REQ-6: Per-seccomp: default sandbox profiles MUST list `getpmsg`/`putpmsg` in DENY/RETURN_ENOSYS rules.

REQ-7: ABI evolution: the kernel SHALL NOT add real STREAMS semantics here. Any future use of this slot SHALL be a NEW syscall name (and would require LKML consensus + LSB review).

REQ-8: Per-`putpmsg`/`putmsg`/`getmsg` family: SVR4 STREAMS had a set; Linux reserves all four numbers via sys_ni_syscall (numbers 181-184 on x86_64; this doc covers only 181).

REQ-9: Per-strace / per-ftrace: the call appears as `getpmsg` in tracing output (mapping syscall-nr 181 to a recognizable name) so observability tooling can distinguish probe attempts from genuine "syscall failed" entries.

REQ-10: Per-`/proc/[pid]/syscall`: if the calling thread is sleeping in this syscall (impossible: ENOSYS is synchronous), the readout would show syscall 181 with all zeroed register snapshots; in practice the syscall never blocks.

REQ-11: Per-`CONFIG_COMPAT`: 32-bit compat entry (`__NR_compat_getpmsg` = 188 for i386 personality on x86_64) routes to the same ENOSYS stub. There is NO architecture-specific 32-bit STREAMS handling.

REQ-12: Per-ABI-policy: the kernel ABI guarantee that "a working syscall does not regress" does NOT apply here — there has never been a working `getpmsg`; ENOSYS is the documented behavior. New kernels SHALL preserve this ENOSYS contract.

REQ-13: Per-LSB compliance: the Linux Standard Base specifies that `getpmsg` returns ENOSYS; this is a documented LSB conformance point.

REQ-14: Per-error-distinguishability: `ENOSYS` is the canonical "syscall not implemented" errno; userspace MUST NOT confuse it with `EOPNOTSUPP` (operation not supported on this object) or `EINVAL` (bad argument). The strict ENOSYS contract preserves this distinction.

## Acceptance Criteria

- [ ] AC-1: `syscall(__NR_getpmsg, ...)` returns `-ENOSYS` on every architecture and every build configuration.
- [ ] AC-2: Audit record per call attempt with syscall-number and caller real-uid.
- [ ] AC-3: Seccomp default profile (systemd / Docker / runc / gVisor) DENIES the syscall.
- [ ] AC-4: Hardened build (grsec) emits ANOM_LEGACY_SYSCALL record per attempt.
- [ ] AC-5: ABI-fuzzer that walks syscall numbers does not crash or leak data through this entry.
- [ ] AC-6: kallsyms / /proc/kallsyms does NOT expose any `sys_getpmsg` symbol (handler is sys_ni_syscall).

## Architecture

```rust
#[syscall(nr = 181, abi = "sysv")]
pub fn sys_getpmsg(_a0: u64, _a1: u64, _a2: u64, _a3: u64, _a4: u64) -> isize {
    AuditLog::record_deprecated_syscall_attempt("getpmsg", 0);
    -ENOSYS as isize
}
```

The implementation is intentionally minimal:

- No argument copying (avoids exposing copy paths to a syscall that should not exist).
- Audit emitted BEFORE returning so attempts are recorded even when the caller does not check errno.
- Returns ENOSYS directly; does NOT consult sys_ni_syscall machinery so the audit hook fires reliably.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_enosys` | INVARIANT | every call ⟹ ENOSYS. |
| `no_arg_copy` | INVARIANT | no `copy_from_user` performed. |
| `audit_pre_return` | INVARIANT | audit hook fires before ENOSYS return. |
| `no_kallsym_handler` | INVARIANT | sys_getpmsg not exported via kallsyms. |

### Layer 2: TLA+

`kernel/sys-ni-getpmsg.tla`:
- States: per-entry, per-audit, per-return.
- Properties:
  - `safety_no_side_effect` — call modifies no kernel state beyond audit ring.
  - `safety_enosys_invariant` — return value is always ENOSYS.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_getpmsg` post: returns ENOSYS unconditionally | `sys_getpmsg` |
| Audit emitted regardless of args | `AuditLog::record_deprecated_syscall_attempt` |

### Layer 4: Verus / Creusot functional

Trivial: function is constant-return `-ENOSYS` with audit side-effect. ABI-stability test: syscall_table[181] resolves to a function returning ENOSYS on all archs.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getpmsg(2)` reinforcement:

- **Per-ENOSYS invariant** — defense against per-handler-revival.
- **Per-no-arg-copy** — defense against per-pointer-deref smuggling via unused-syscall arg.
- **Per-audit-on-attempt** — defense against per-legacy-vector reconnaissance.
- **Per-no-kallsym-symbol** — defense against per-handler-target ROP.
- **Per-seccomp-default-deny** — defense against per-legacy-syscall in unintended sandbox.

## Grsecurity / PaX surface

- **GRKERNSEC_DEPRECATED_SYSCALL_ENOSYS_STRICT** — getpmsg(2) returns `-ENOSYS` unconditionally; the call path emits an audit record (ANOM_DEPRECATED_SYSCALL) with caller comm/uid. Defense against per-syscall-table reconnaissance / unimplemented-slot probing (which malware sometimes uses to fingerprint kernel build).
- **PaX UDEREF / SMAP** — the syscall path performs NO user-pointer copy, so UDEREF/SMAP are not exercised; the audit record captures the raw register values without dereferencing them.
- **GRKERNSEC_HIDESYM on syscall-table** — `sys_call_table` is not exposed; `sys_ni_syscall` and any specific `sys_getpmsg` entry is not resolvable via kallsyms for non-root callers. Defense against per-syscall-table ROP-gadget discovery.
- **GRKERNSEC_AUDIT_LEGACY_SYSCALL** — kernel.audit emits ANOM_LEGACY_GETPMSG record per call attempt with caller uid/comm/syscall-nr. SIEM correlation across the legacy-syscall family.
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family policy** — getpmsg(2), putpmsg(2), sysfs(2), ustat(2), uselib(2), and other unimplemented stubs share a common audit record type so SIEM rules can match the family.
- **GRKERNSEC_LOG_RATE_LIMIT** — defense against per-ENOSYS log flood from repeated syscall-table walking malware.
- **PaX KERNEXEC verification** — `sys_getpmsg` entry resolves to a W^X-compliant function (read-only text); defense against per-text-patch attempts targeting the slot.
- **PAX_RAP on the syscall-table entry** — RAP tag prevents an attacker from redirecting the slot to a chosen-ROP gadget; the slot must resolve to a function with the syscall-return RAP signature.
- **GRKERNSEC_SECCOMP_DEFAULT_DENY** — distro hardened profiles ship a seccomp filter that returns RET_ERRNO ENOSYS for getpmsg/putpmsg pre-entry; defense in depth on top of the kernel's own ENOSYS.
- **PaX MEMORY_SANITIZE** — no stack buffer is touched by this syscall, so MEMORY_SANITIZE is vacuously satisfied; documented for completeness.
- **GRKERNSEC_PROC_GETPID consistency** — audit record carries caller pid in caller's namespace for cross-correlation with /proc/[pid]/ records.
- **GRKERNSEC_AUDIT_NI_SYSCALL** — every call to a sys_ni_syscall slot (not just getpmsg) emits a numbered audit record; defense against per-table-scan reconnaissance via a class-wide hook.
- **GRKERNSEC_BRUTE compatibility** — repeated getpmsg(2) attempts from the same uid trip GRKERNSEC_BRUTE pid-level brute-force detection (with shared cooldown across the legacy-syscall family); defense against per-fingerprint scanning.

## Open Questions

- (none at this Tier-5 level — see "Out of Scope" below for related items.)

## Out of Scope

- SVR4 STREAMS semantics (out of Linux kernel scope; never implemented).
- `putpmsg(2)`, `putmsg(2)`, `getmsg(2)` — covered separately as part of the unimplemented-stub family if expanded.
- `sys_ni_syscall` generic stub (covered in Tier-3 `kernel/sys.md` if expanded).
- LSB conformance tooling (out of kernel scope).
- Per-arch syscall_table layout details (covered in arch Tier-3 docs).
- Per-`sys_call_table` ROP-defense (covered in arch Tier-3 hardening doc).
- Per-seccomp default-profile composition (out of kernel scope; userspace policy).
- Implementation code.
