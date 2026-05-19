# Tier-5 syscall: putpmsg(2) — syscall 182 (NEVER IMPLEMENTED)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys_ni.c (COND_SYSCALL(putpmsg))
  - arch/x86/entry/syscalls/syscall_64.tbl (182  64  putpmsg)
  - net/socket.c (mainline replacement: socket API + sendmsg/recvmsg)
-->

## Summary

`putpmsg(2)` is a **never-implemented, permanently-stubbed** syscall whose slot was reserved in the Linux x86_64 syscall table for binary compatibility with SVR4-era STREAMS APIs. SVR4 (System V Release 4) and its descendants (Solaris, AIX, HP-UX, UnixWare) used the STREAMS framework for protocol-stack composition; `getmsg(2)`, `putmsg(2)`, `getpmsg(2)` ("priority" variants), and `putpmsg(2)` were the userspace data-path primitives, sending/receiving control-plus-data messages over a STREAMS file descriptor with optional band priority. Linux never adopted STREAMS — the mainline socket / sendmsg / recvmsg / TIPC / Netlink stack covered the same use cases without STREAMS' modular-driver chain — but the slot was reserved so that distributors / ABI translators could implement STREAMS in userspace or via a future LKM without renumbering.

The slot has never been bound to any kernel-side implementation in mainline. It returns `-ENOSYS` unconditionally via `COND_SYSCALL(putpmsg)`. Companion slot `getpmsg(2)` (#181) is identically stubbed; mainline `putmsg(2)` / `getmsg(2)` were not even reserved.

Critical for: ABI-table completeness, SVR4-binary-compatibility audit, intrusion-detection (any putpmsg caller is anomalous), grsec deprecated-syscall audit.

## Signature

```c
/* No mainline prototype — the slot was reserved but never defined.
 * SVR4 historical:
 */
int putpmsg(int fildes, const struct strbuf *ctlptr,
            const struct strbuf *dataptr, int band, int flags);

struct strbuf {
    int   maxlen;   /* unused on put */
    int   len;      /* number of bytes in buf */
    char *buf;      /* pointer to data */
};
```

```c
/* Mainline (every Linux kernel ever) effective prototype: stub */
long putpmsg(int fd, const struct strbuf __user *ctlptr,
             const struct strbuf __user *dataptr, int band,
             int flags);  /* always -ENOSYS */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in (ignored) | SVR4: STREAMS file descriptor. |
| `ctlptr` | `const struct strbuf __user *` | in (ignored) | SVR4: control-part message; NULL = data-only. |
| `dataptr` | `const struct strbuf __user *` | in (ignored) | SVR4: data-part message; NULL = control-only. |
| `band` | `int` | in (ignored) | SVR4: priority band (0..255). |
| `flags` | `int` | in (ignored) | SVR4: MSG_HIPRI / MSG_BAND / MSG_ANY. |

All arguments are unread by the stub.

## Return value

| Value | Meaning |
|---|---|
| `-1` + `errno = ENOSYS` | Always. |

## Errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always. |

## ABI surface

```text
__NR_putpmsg (x86_64)    = 182    /* slot reserved, never implemented */
__NR_putpmsg (i386)      = 189    /* slot reserved, never implemented */
__NR_getpmsg (x86_64)    = 181    /* companion slot, also stubbed */
__NR_putpmsg (arm64)     = N/A    /* never assigned */
__NR_putpmsg (riscv)     = N/A
__NR_putpmsg (loongarch) = N/A

/* sys_ni.c:
 *     COND_SYSCALL(getpmsg);
 *     COND_SYSCALL(putpmsg);
 */
```

## Compatibility contract

REQ-1: Syscall number is **182** on x86_64; **189** on i386. Numbers reserved-forever for SVR4 binary-compat headroom.

REQ-2: Every invocation MUST return `-ENOSYS`. No argument validation, no `copy_from_user` of the strbuf control or data parts, no LSM hook from the stub itself.

REQ-3: The stub MUST NOT consult `current_cred()`, MUST NOT dereference `ctlptr` / `dataptr`, MUST NOT examine `fd`, MUST NOT validate `flags` or `band`.

REQ-4: SVR4 `flags` semantics (`MSG_HIPRI`, `MSG_BAND`, `MSG_ANY`) MUST NOT be honoured even partially.

REQ-5: On architectures that never assigned the number, `-ENOSYS` originates at the syscall-table boundary.

REQ-6: Modern userspace MUST use sockets:
- `socket(AF_INET, SOCK_STREAM, ...)` / `socket(AF_UNIX, ...)` / `socket(AF_NETLINK, ...)` + `sendmsg(2)` / `recvmsg(2)` for protocol-stack message flow.
- `SO_PRIORITY` / `IP_TOS` / TIPC for priority-banded delivery.
- `mq_send(2)` / `mq_receive(2)` for POSIX message queues if classical message-queue semantics are required.

REQ-7: `seccomp` filters MAY allowlist or denylist syscall 182 explicitly.

REQ-8: `ptrace` of the stub sees syscall-entry/exit pair with return `-ENOSYS`.

REQ-9: `sys_putpmsg` is NOT a distinct kallsyms entry; the slot resolves to `__x64_sys_ni_syscall`.

REQ-10: ABI-stability: slot 182 MUST NOT be reassigned.

REQ-11: Per-Rookery: legacy slot reserved at compile time; dispatches to shared `sys_ni::enosys()`. No STREAMS framework exists in Rookery.

## Acceptance Criteria

- [ ] AC-1: `syscall(SYS_putpmsg, fd, NULL, NULL, 0, 0)` returns `-1, ENOSYS`.
- [ ] AC-2: `syscall(182, fd, &ctl, &data, 7, 0)` returns `-1, ENOSYS`; ctl/data unread.
- [ ] AC-3: Stub returns `-ENOSYS` regardless of whether `fd` is valid.
- [ ] AC-4: Stub returns `-ENOSYS` regardless of capabilities.
- [ ] AC-5: `seccomp` filter denying `putpmsg` produces configured action.
- [ ] AC-6: `strace -e putpmsg` traces one `putpmsg(...) = -1 ENOSYS`.
- [ ] AC-7: Architectures without the slot return `-ENOSYS` via the table boundary.
- [ ] AC-8: Grsec-enabled kernel logs each invocation under `GRKERNSEC_DEPRECATED_SYSCALL_AUDIT`.
- [ ] AC-9: Boot-time sys_call_table integrity check confirms slot 182 = `sys_ni_syscall`.
- [ ] AC-10: 1e6 calls under stress allocate no kernel memory and acquire no lock.
- [ ] AC-11: Mainline socket APIs (`sendmsg`, `recvmsg`) continue to work normally.
- [ ] AC-12: Companion slot `getpmsg(2)` (#181) also returns `-ENOSYS`.

## Architecture

```rust
#[syscall(nr = 182, abi = "sysv", legacy = true, never_implemented = true)]
pub fn sys_putpmsg(
    _fd: i32,
    _ctlptr: UserPtr<Strbuf>,
    _dataptr: UserPtr<Strbuf>,
    _band: i32,
    _flags: i32,
) -> isize {
    SysNi::enosys("putpmsg")
}

#[syscall(nr = 181, abi = "sysv", legacy = true, never_implemented = true)]
pub fn sys_getpmsg(
    _fd: i32,
    _ctlptr: UserPtr<Strbuf>,
    _dataptr: UserPtr<Strbuf>,
    _bandp: UserPtr<i32>,
    _flagsp: UserPtr<i32>,
) -> isize {
    SysNi::enosys("getpmsg")
}
```

`SysNi::enosys(name: &'static str) -> isize`:
1. /* Shared no-op return path */
2. #[cfg(feature = "grsec_deprecated_audit")]
3.   GrsecAudit::log_legacy_syscall(name, current_creds());
4. -ENOSYS

`GrsecAudit::log_legacy_syscall(name, creds)`:
1. let task = current();
2. let event = LegacySyscallEvent {
3.   name,
4.   pid: task.tgid_in(init_pid_ns()),
5.   uid: creds.euid,
6.   exe: task.exe_file_path(),
7.   timestamp: now_monotonic_ns(),
8. };
9. AuditQueue::enqueue(event);

`SysCallTable::verify_legacy_slots()` boot-time:
1. for (nr, _) in LEGACY_STUB_SLOTS {
2.   assert!(sys_call_table[nr] == sys_ni_syscall as *const _);
3. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_enosys` | INVARIANT | per-call: return value is exactly `-ENOSYS`. |
| `no_strbuf_deref` | INVARIANT | per-stub: ctlptr/dataptr never dereferenced. |
| `no_fd_lookup` | INVARIANT | per-stub: no `fget` / `fdget`. |
| `no_user_copy_in_or_out` | INVARIANT | per-stub: no copy_from_user / copy_to_user. |
| `no_cap_check` | INVARIANT | per-stub: no `capable(...)` call. |
| `no_alloc` | INVARIANT | per-stub: no kmalloc / vmalloc. |

### Layer 2: TLA+

`kernel/legacy-syscalls.tla`:
- States: per-syscall-table slot binding, per-stub invocation.
- Properties:
  - `safety_slot_182_bound_to_ni` — per-boot: slot 182 → sys_ni_syscall.
  - `safety_slot_181_bound_to_ni` — per-boot: companion slot 181 → sys_ni_syscall.
  - `safety_stub_returns_enosys` — per-call: ret == -ENOSYS.
  - `safety_no_fd_table_touch` — per-call: file descriptor table invariant.
  - `liveness_terminates` — per-call: O(1) steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_putpmsg` post: ret == -ENOSYS | `sys_putpmsg` |
| `sys_getpmsg` post: ret == -ENOSYS | `sys_getpmsg` |
| `sys_putpmsg` post: no observable state mutation (except audit log) | `sys_putpmsg` |
| `sys_putpmsg` post: fd table unchanged | `sys_putpmsg` |

### Layer 4: Verus / Creusot functional

Per-mainline-policy: the slot has never resolved to an implementation. Equivalence with `sys_ni_syscall`. SVR4 STREAMS semantics are not provided.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`putpmsg(2)` reinforcement:

- **Per-stub no-strbuf-deref** — defense against per-buffer-pointer fault on stale or hostile pointers.
- **Per-stub no-fd-lookup** — defense against per-fd-table contention by legacy-syscall flood.
- **Per-stub no-user-write** — defense against per-userspace-corruption via stale buffer pointer.
- **Per-stub no-alloc** — defense against per-DoS via legacy-syscall flood.
- **Per-stub constant-time** — defense against per-side-channel timing probe.
- **Per-syscall-table-integrity** — boot-time + periodic verification slot 182 → sys_ni_syscall.
- **Per-seccomp filterable** — userspace can block the slot.
- **Per-socket gating intact** — sendmsg/recvmsg LSM hooks (`security_socket_sendmsg`) are the only data-path gates; the stub cannot bypass.

## Grsecurity / PaX-style Reinforcement

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every invocation logged with `pid`, `uid`, `comm`, `exe path`, `fd` value (captured from syscall-entry tracepoint), `flags`, timestamp. Rate-limited per-uid. Extremely high signal: STREAMS has NEVER existed in Linux.
- **Boot-time sys_call_table verification** — slot 182 (and companion 181) asserted `sys_ni_syscall`; mismatch panics. A rootkit that binds these slots to a covert message-injection primitive is blocked at boot.
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family** — optional stricter mode: invocation triggers `SIGSYS` to the caller. Recommended in hardened distributions; no legitimate Linux software has EVER used these slots.
- **GRKERNSEC_PROC_USERGROUP** — STREAMS pseudo-devices do not exist; there is no `/dev/streams/` path for an attacker to exploit.
- **PaX KERNEXEC on syscall-table page** — slot-hook attempts fault.
- **PAX_RANDKSTACK at stub entry** — randomizes kernel-stack offset per call.
- **No capability elevation possible** — stub does not consult capabilities; confused-deputy escalation impossible.
- **GRKERNSEC_HIDESYM symbols** — `sys_putpmsg`, `sys_getpmsg` are not distinct kallsyms entries.
- **Audit ring-buffer flood-immune** — per-uid summary record at 1 Hz, not per-call.
- **Sockets-only enforced** — protocol-stack data path is via sockets + LSM hooks; the legacy slot cannot provide an alternate channel.
- **Kexec preservation** — legacy-slot binding rebuilt identically on post-kexec kernel.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `sendmsg(2)` / `recvmsg(2)` — mainline replacement (Tier-5 separate docs).
- `mq_send(2)` / `mq_receive(2)` — POSIX message queues (Tier-5 separate docs).
- Netlink sockets — kernel↔userspace control plane.
- The SVR4 STREAMS framework itself (not part of Rookery / Linux).
- `sys_ni_syscall` shared stub (Tier-3 `kernel/sys_ni.md`).
- Grsec audit subsystem (Tier-3 `security/grsec/audit.md`).
- Implementation code.
