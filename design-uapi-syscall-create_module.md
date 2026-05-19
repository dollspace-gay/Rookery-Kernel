---
title: "Tier-5 syscall: create_module(2) — syscall 174 (DEPRECATED)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`create_module(2)` is a **deprecated, permanently-stubbed** syscall whose original Linux 1.x / 2.0 role was to allocate kernel-side memory for a loadable kernel module before the userspace tool (`insmod` of the pre-1996 era) wrote relocated text into it via `init_module(2)`. The two-step model (`create_module` → `init_module`) was replaced by the single-step `init_module(2)` interface in Linux 2.6 (2003), and again by `finit_module(2)` in 2012. Since then `create_module` exists ONLY as a numbered slot in the syscall table so that the syscall-number ABI remains stable; every invocation returns `-ENOSYS`.

This doc covers the **stub** path — the `COND_SYSCALL(create_module)` weak-alias that returns `-ENOSYS`. It is documented for ABI-stability auditing, security-monitoring (any process invoking syscall 174 is anomalous on a modern kernel), and to specify the hardening behaviour required when Rookery emulates the legacy slot.

Critical for: ABI-table completeness, syscall-table integrity verification, intrusion-detection signatures (legacy-syscall scanning), grsec deprecated-syscall audit policy, defence against module-loading downgrade attacks that probe the legacy slot.

### Acceptance Criteria

- [ ] AC-1: `syscall(SYS_create_module, "foo", 0)` on x86_64 returns `-1` with `errno == ENOSYS`.
- [ ] AC-2: `syscall(174, NULL, 0)` returns `-1, ENOSYS` — NULL pointer is not dereferenced.
- [ ] AC-3: `syscall(174, ...)` as root with `CAP_SYS_MODULE` still returns `-ENOSYS`.
- [ ] AC-4: `syscall(174, ...)` with `modules_disabled = 0` still returns `-ENOSYS` (state does not matter).
- [ ] AC-5: `seccomp` filter denying `create_module` produces the filter's configured action; stub does not run.
- [ ] AC-6: `strace -e create_module` on the call traces exactly one `create_module(...) = -1 ENOSYS`.
- [ ] AC-7: `/proc/<pid>/syscall` during a synthetic block on the stub does not show `create_module` because the stub never sleeps.
- [ ] AC-8: Architectures without the slot (arm64) return `-ENOSYS` via the syscall-table boundary; the entry trampoline does not reach Rookery's stub function.
- [ ] AC-9: Grsec-enabled kernel logs each invocation under `GRKERNSEC_DEPRECATED_SYSCALL_AUDIT` (see hardening).
- [ ] AC-10: Boot-time sys_call_table integrity check confirms slot 174 resolves to `sys_ni_syscall` (Rookery refuses to boot otherwise under grsec).
- [ ] AC-11: Repeated invocation under stress (1e6 calls) does not allocate kernel memory or take any lock.
- [ ] AC-12: Module-loading via `init_module(2)` and `finit_module(2)` works normally regardless of `create_module` invocation history.

### Architecture

```rust
#[syscall(nr = 174, abi = "sysv", legacy = true)]
pub fn sys_create_module(_name: UserPtr<u8>, _size: usize) -> isize {
    SysNi::enosys("create_module")
}
```

`SysNi::enosys(name: &'static str) -> isize`:
1. /* Single shared no-op return path for every stubbed legacy syscall */
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

`SysCallTable::verify_legacy_slots()` (boot-time):
1. for (nr, expected) in LEGACY_STUB_SLOTS {
2.   if sys_call_table[nr as usize] != sys_ni_syscall as *const _ {
3.     panic!("syscall table corruption: slot {} = {:p}, expected sys_ni_syscall",
4.            nr, sys_call_table[nr as usize]);
5.   }
6. }

### Out of Scope

- `init_module(2)` — modern module loader (Tier-5 separate doc).
- `finit_module(2)` — fd-based module loader (Tier-5 separate doc).
- `delete_module(2)` — module removal (Tier-5 separate doc, exists).
- Module loader internals (Tier-3 `kernel/module/main.md`).
- `sys_ni_syscall` shared stub implementation (Tier-3 `kernel/sys_ni.md`).
- Grsec audit subsystem (Tier-3 `security/grsec/audit.md`).
- Implementation code.

### signature

```c
/* Pre-2.6 (historical) prototype, retained ONLY for headers */
caddr_t create_module(const char *name, size_t size);
```

```c
/* Modern (Linux 2.6+) effective prototype: stub */
long create_module(const char __user *name, size_t size);  /* always -ENOSYS */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `name` | `const char __user *` | in (ignored) | Historical: NUL-terminated module name. Modern stub: unread. |
| `size` | `size_t` | in (ignored) | Historical: module text+data size to reserve. Modern stub: unread. |

The modern kernel does **not** dereference either argument; the stub returns `-ENOSYS` before any `copy_from_user` would occur, so a bad pointer for `name` does not produce `-EFAULT`.

### return value

| Value | Meaning |
|---|---|
| `-1` + `errno = ENOSYS` | Always. The syscall is removed; only the number is reserved. |

There is no success path. Userspace MUST treat the slot as permanently unavailable. Tools (`modprobe`, `kmod`, `insmod`) detect this at build time and use `init_module(2)` / `finit_module(2)` instead.

### errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always. Returned unconditionally from `sys_ni_syscall`. |

### abi surface

```text
__NR_create_module (x86_64)   = 174    /* slot reserved, stubbed */
__NR_create_module (i386)     = 127    /* slot reserved, stubbed */
__NR_create_module (arm64)    = N/A    /* never assigned on generic-syscall */
__NR_create_module (riscv)    = N/A
__NR_create_module (loongarch)= N/A

/* The slot is x86-legacy only. arm64 / riscv / loongarch / generic-syscall
 * architectures never assigned a number; trying to invoke #174 there yields
 * -ENOSYS at the syscall-table boundary, not from a stub.
 */

/* sys_ni.c provides:
 *     COND_SYSCALL(create_module);
 * which compiles to a weak alias of sys_ni_syscall returning -ENOSYS.
 */
```

### compatibility contract

REQ-1: Syscall number is **174** on x86_64; **127** on i386. The numbers are **reserved-forever**: they will never be reassigned to a different syscall, even though the implementation is permanently stubbed.

REQ-2: Every invocation MUST return `-ENOSYS`. No argument validation, no `copy_from_user`, no LSM hook, no audit emission from the stub path (audit subsystem may still log the syscall-entry event via syscall-entry tracepoint).

REQ-3: The stub MUST NOT consult `current_cred()`, `capable(CAP_SYS_MODULE)`, or `modules_disabled`. It returns `-ENOSYS` even for root in init_user_ns.

REQ-4: The stub MUST NOT honour module-loading flags (`MODULE_INIT_*`). Userspace MUST use `init_module(2)` (syscall 175) or `finit_module(2)` (syscall 313) instead.

REQ-5: On architectures that never assigned the number (arm64, riscv, loongarch, generic-syscall), the return value `-ENOSYS` originates from the syscall-table boundary (`sys_call_table[__NR_NR]` is `sys_ni_syscall`), not from a per-syscall stub. Userspace observes the same `-ENOSYS` either way.

REQ-6: Modern userspace (`kmod` ≥ 5, `module-init-tools` ≥ 3.0) MUST NOT call this syscall. Calling it is a strong indicator of either (a) a binary >25 years old, (b) a rootkit probing for legacy loader paths, or (c) an exploit verifying the stub returns the documented errno.

REQ-7: `seccomp` filters MAY allowlist or denylist syscall 174 explicitly; the kernel does not exempt the stub from seccomp filtering.

REQ-8: `ptrace` of the stub sees the syscall-entry/exit pair with return value `-ENOSYS`. `strace -e trace=%modules` includes `create_module` for completeness.

REQ-9: Per-`/proc/kallsyms`: `sys_create_module` is NOT exported as a distinct symbol; the slot resolves to `__x64_sys_ni_syscall` (or arch-equivalent). Tooling MUST NOT pattern-match on `sys_create_module`.

REQ-10: ABI-stability: the kernel MUST NOT reassign syscall 174 to any new functionality. The number is permanently burned. This guarantee is enforced by the syscall-table ABI policy.

REQ-11: Per-Rookery: the legacy slot is reserved at compile time; the implementation dispatches to a single shared `sys_ni::enosys()` return path; an optional audit hook fires (grsec policy below).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_enosys` | INVARIANT | per-create_module call: return value is exactly `-ENOSYS`. |
| `no_user_deref` | INVARIANT | per-create_module stub: no `copy_from_user` invocation. |
| `no_cap_check` | INVARIANT | per-create_module stub: no `capable(...)` call. |
| `no_alloc` | INVARIANT | per-create_module stub: no kmalloc / vmalloc. |
| `no_lock_taken` | INVARIANT | per-create_module stub: no kernel lock acquired. |
| `idempotent_stub` | INVARIANT | per-N calls: return value identical (`-ENOSYS`). |

### Layer 2: TLA+

`kernel/legacy-syscalls.tla`:
- States: per-syscall-table slot binding, per-stub invocation, per-audit-queue enqueue.
- Properties:
  - `safety_slot_174_bound_to_ni` — per-boot: slot 174 → sys_ni_syscall.
  - `safety_stub_returns_enosys` — per-call: ret == -ENOSYS.
  - `safety_no_implementation` — per-call: no implementation path executes (only audit + return).
  - `liveness_terminates` — per-call: stub returns in O(1) steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_create_module` post: ret == -ENOSYS | `sys_create_module` |
| `sys_create_module` post: no side-effects on kernel state (except audit log) | `sys_create_module` |
| `verify_legacy_slots` post: every legacy slot is sys_ni_syscall | `SysCallTable::verify_legacy_slots` |

### Layer 4: Verus / Creusot functional

Per-`create_module(2)` man-page (NOTES: "This system call is present only in kernels before Linux 2.6.") equivalence with `sys_ni_syscall`. `strace` output equivalence verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`create_module(2)` reinforcement:

- **Per-stub no-arg-deref** — defense against per-stale-userspace pointer fault (pointers never read).
- **Per-stub no-alloc** — defense against per-DoS via legacy-syscall flood.
- **Per-stub constant-time** — defense against per-side-channel timing probe.
- **Per-stub no-lock** — defense against per-lock-flood DoS.
- **Per-syscall-table-integrity** — boot-time + periodic verification slot 174 points at sys_ni_syscall.
- **Per-seccomp filterable** — userspace can block the slot explicitly without kernel-side carve-out.

### grsecurity / pax-style reinforcement

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every invocation of syscall 174 is logged with `pid`, `uid`, `comm`, `exe path`, `argv` snapshot, and timestamp. Rate-limited per-uid to defeat log-flood while preserving forensic value. Enabled by default in hardened builds.
- **Boot-time sys_call_table verification** — at boot and on every kexec, the kernel walks `sys_call_table[174]` and asserts it is `sys_ni_syscall`. Mismatch panics the kernel (corruption / rootkit indicator).
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family** — optional stricter mode: invocation of any deprecated-stub syscall (174, 177, 178, 180, 156, 236, 183, 182) triggers `SIGSYS` to the caller instead of returning `-ENOSYS`. Useful for hardened distributions where no legitimate process needs the legacy slots.
- **PaX KERNEXEC on syscall-table page** — `sys_call_table` mapped read-only after init; any write attempt (rootkit hooking #174 to bypass init_module hardening) faults.
- **PAX_RANDKSTACK at stub entry** — even the stub randomizes kernel-stack offset; defeats per-syscall-stack-fingerprint reconnaissance via high-frequency #174 calls.
- **GRKERNSEC_PROC_USERGROUP** — `/proc/<pid>/syscall` shows the stub entry only to the task itself / CAP_SYS_PTRACE holders.
- **No CAP_SYS_MODULE elevation possible** — the stub is deliberately written to NOT consult capabilities; a confused-deputy escalation pattern is impossible.
- **Per-audit ring-buffer immune to flood** — repeated #174 from a single uid produces one summary record per second, not per call.
- **GRKERNSEC_HIDESYM** — `sys_create_module` symbol is not exposed in `/proc/kallsyms` (it does not exist as a distinct symbol; the slot resolves to `sys_ni_syscall`, which is also hidden under HIDESYM).
- **Kexec preservation** — the legacy-slot binding is rebuilt identically on the post-kexec kernel; an attacker cannot use kexec to swap slot 174 to a malicious target.

