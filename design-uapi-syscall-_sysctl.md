---
title: "Tier-5 syscall: _sysctl(2) — syscall 156 (REMOVED)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`_sysctl(2)` is a **removed, permanently-stubbed** syscall whose original Linux 1.3 / 2.0 role was to read/write kernel tunables via a binary `name[]` integer-path (e.g. `{CTL_KERN, KERN_HOSTNAME}`) and a typed buffer. It was BSD's `sysctl(3)` interface ported to Linux. The interface was always considered a misfeature: the binary `ctl_name` integers had to be allocated centrally and were churned constantly; the parallel text-path `/proc/sys/` interface (added almost immediately afterwards) was always the canonical path. The man page warned **for decades** "Glibc does not provide a wrapper for this system call; call it using syscall(2). But: don't! See NOTES." In Linux 5.5 (2020) the entire implementation was removed (`commit 61a47c1ad3a4d`); the slot now returns `-ENOSYS`.

Critical for: ABI-table completeness, intrusion-detection (legacy binary-sysctl probe), grsec deprecated-syscall audit, defence against sysctl-binary-path downgrade probing.

### Acceptance Criteria

- [ ] AC-1: `syscall(SYS__sysctl, &args)` with valid `args` returns `-1, ENOSYS`.
- [ ] AC-2: `syscall(156, NULL)` returns `-1, ENOSYS`; no pointer dereference.
- [ ] AC-3: `syscall(156, ...)` as root returns `-1, ENOSYS`.
- [ ] AC-4: Reading `/proc/sys/kernel/hostname` continues to work (the modern path is unaffected).
- [ ] AC-5: Writing `/proc/sys/kernel/hostname` continues to work (gated by `CAP_SYS_ADMIN`).
- [ ] AC-6: `seccomp` filter denying `_sysctl` produces configured action.
- [ ] AC-7: `strace -e _sysctl` traces one `_sysctl(...) = -1 ENOSYS`.
- [ ] AC-8: Architectures without the slot return `-ENOSYS` via the table boundary.
- [ ] AC-9: Grsec-enabled kernel logs each invocation under `GRKERNSEC_DEPRECATED_SYSCALL_AUDIT`.
- [ ] AC-10: Boot-time sys_call_table integrity check confirms slot 156 = `sys_ni_syscall`.
- [ ] AC-11: 1e6 calls under stress allocate no kernel memory and acquire no lock.
- [ ] AC-12: Compat (32-bit on 64-bit) invocation also returns `-ENOSYS`.

### Architecture

```rust
#[syscall(nr = 156, abi = "sysv", legacy = true)]
pub fn sys_sysctl(_args: UserPtr<SysctlArgs>) -> isize {
    SysNi::enosys("_sysctl")
}

#[syscall(nr = 156, abi = "compat", legacy = true)]
pub fn compat_sys_sysctl(_args: UserPtr<CompatSysctlArgs>) -> isize {
    SysNi::enosys("_sysctl(compat)")
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
4. assert!(compat_sys_call_table[156] == compat_sys_ni_syscall as *const _);

### Out of Scope

- `/proc/sys/` virtual filesystem (Tier-3 `fs/proc/sysctl.md`).
- `kernel/sysctl.c` registered-tunable tree (Tier-3 `kernel/sysctl.md`).
- LSM `security_sysctl` hook (Tier-3 `security/lsm-hooks.md`).
- BSD `sysctl(3)` interface (not provided).
- `sys_ni_syscall` shared stub (Tier-3 `kernel/sys_ni.md`).
- Grsec audit subsystem (Tier-3 `security/grsec/audit.md`).
- Implementation code.

### signature

```c
/* Pre-5.5 historical prototype, retained ONLY for headers */
int _sysctl(struct __sysctl_args *args);

struct __sysctl_args {
    int    *name;        /* integer name vector */
    int     nlen;        /* length of name vector */
    void   *oldval;      /* 0 or address where to store old value */
    size_t *oldlenp;     /* available room for old value, overwritten by actual size */
    void   *newval;      /* 0 or address of new value */
    size_t  newlen;      /* size of new value */
    unsigned long __unused[4];
};
```

```c
/* Modern (Linux 5.5+) effective prototype: stub */
long _sysctl(struct __sysctl_args __user *args);  /* always -ENOSYS */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `args` | `struct __sysctl_args __user *` | in (ignored) | Historical: aggregate request struct. Modern stub: unread. |

### return value

| Value | Meaning |
|---|---|
| `-1` + `errno = ENOSYS` | Always (Linux ≥ 5.5). |

Historically (Linux ≤ 5.4), success returned 0 and the oldval/oldlenp pair was filled.

### errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always on Linux ≥ 5.5. |

### abi surface

```text
__NR__sysctl  (x86_64)   = 156    /* slot reserved, stubbed */
__NR__sysctl  (i386)     = 149    /* slot reserved, stubbed */
__NR__sysctl  (arm64)    = N/A    /* never assigned on generic-syscall */
__NR__sysctl  (riscv)    = N/A
__NR__sysctl  (loongarch)= N/A

/* sys_ni.c:
 *     COND_SYSCALL(sysctl);
 *     COND_SYSCALL_COMPAT(sysctl);
 *
 * Note: the leading underscore in the userspace name is glibc's convention;
 * the kernel symbol was `sys_sysctl` historically.
 */
```

### compatibility contract

REQ-1: Syscall number is **156** on x86_64; **149** on i386. Numbers reserved-forever.

REQ-2: Every invocation MUST return `-ENOSYS`. No argument validation, no `copy_from_user`, no `sysctl` table walk, no LSM hook from the stub itself.

REQ-3: The stub MUST NOT consult `current_cred()`, capability bits, or any sysctl-permission predicate. It returns `-ENOSYS` even for root.

REQ-4: The stub MUST NOT consult the `kernel/sysctl.c` registered-tunable tree. That tree remains alive and queryable via `/proc/sys/`.

REQ-5: On architectures that never assigned the number (arm64, riscv, loongarch), `-ENOSYS` originates at the syscall-table boundary.

REQ-6: Modern userspace MUST use `/proc/sys/<path>` (e.g. `/proc/sys/kernel/hostname`) for tunable read/write. POSIX `sysctl(3)` from BSD is not provided.

REQ-7: `seccomp` filters MAY allowlist or denylist syscall 156 explicitly.

REQ-8: `ptrace` of the stub sees syscall-entry/exit pair with return `-ENOSYS`.

REQ-9: `sys_sysctl` is NOT a distinct kallsyms entry; the slot resolves to `__x64_sys_ni_syscall`.

REQ-10: ABI-stability: slot 156 MUST NOT be reassigned. This is a "burned" number.

REQ-11: Compat layer: `compat_sys_sysctl` is similarly stubbed via `COND_SYSCALL_COMPAT(sysctl)`.

REQ-12: Per-Rookery: legacy slot reserved at compile time; dispatches to shared `sys_ni::enosys()`. The Rookery sysctl registry is only reachable via `/proc/sys/`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_enosys` | INVARIANT | per-call: return value is exactly `-ENOSYS`. |
| `no_sysctl_tree_walk` | INVARIANT | per-stub: no traversal of registered sysctl tree. |
| `no_user_copy_in_or_out` | INVARIANT | per-stub: no copy_from_user / copy_to_user. |
| `no_cap_check` | INVARIANT | per-stub: no `capable(...)` call. |
| `no_alloc` | INVARIANT | per-stub: no kmalloc / vmalloc. |
| `compat_consistent` | INVARIANT | per-compat-call: same -ENOSYS return as native. |

### Layer 2: TLA+

`kernel/legacy-syscalls.tla`:
- States: per-syscall-table slot binding (both native and compat), per-stub invocation.
- Properties:
  - `safety_slot_156_bound_to_ni` — per-boot: slot 156 → sys_ni_syscall (both native + compat).
  - `safety_stub_returns_enosys` — per-call: ret == -ENOSYS.
  - `safety_no_tunable_mutation` — per-call: sysctl tree unchanged.
  - `liveness_terminates` — per-call: O(1) steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_sysctl` post: ret == -ENOSYS | `sys_sysctl` |
| `compat_sys_sysctl` post: ret == -ENOSYS | `compat_sys_sysctl` |
| `sys_sysctl` post: sysctl tunable tree unchanged | `sys_sysctl` |
| `sys_sysctl` post: no observable state mutation (except audit log) | `sys_sysctl` |

### Layer 4: Verus / Creusot functional

Per-`_sysctl(2)` man-page ("This system call no longer exists on current kernels.") equivalence with `sys_ni_syscall`. `/proc/sys/` is the canonical interface; equivalence with the modern path is per-Documentation/admin-guide/sysctl/.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`_sysctl(2)` reinforcement:

- **Per-stub no-tunable-disclosure** — defense against per-info-leak via binary-sysctl path.
- **Per-stub no-tunable-mutation** — defense against per-config-poisoning that bypasses `/proc/sys/` LSM hooks.
- **Per-stub no-user-write** — defense against per-userspace-corruption via stale buffer pointer.
- **Per-stub no-alloc** — defense against per-DoS via legacy-syscall flood.
- **Per-stub constant-time** — defense against per-side-channel timing probe.
- **Per-syscall-table-integrity** — boot-time + periodic verification slot 156 → sys_ni_syscall (native + compat).
- **Per-seccomp filterable** — userspace can block the slot.
- **Per-/proc/sys gating intact** — LSM hooks (`security_sysctl`, `selinux_sysctl_set_attr`) fire on the modern path; the stub cannot bypass.

### grsecurity / pax-style reinforcement

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every invocation logged with `pid`, `uid`, `comm`, `exe path`, timestamp. Rate-limited per-uid. The binary-sysctl interface has been removed since 2020; any caller in 2026 is anomalous.
- **Boot-time sys_call_table verification** — slot 156 asserted `sys_ni_syscall` in both native and compat tables; mismatch panics.
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family** — optional stricter mode: invocation triggers `SIGSYS` to the caller. Especially important here because the syscall historically supported `KERN_*` tunables that bypassed `/proc/sys/` LSM hooks; eliminating the slot eliminates an LSM-bypass class.
- **LSM hook coverage forced through `/proc/sys/`** — `security_sysctl` and SELinux's `sysctl_set_attr` are the only gating points; the stub does not provide an alternative.
- **PaX KERNEXEC on syscall-table page** — slot-hook attempts fault.
- **PAX_RANDKSTACK at stub entry** — randomizes kernel-stack offset per call.
- **No CAP_SYS_ADMIN elevation possible** — stub does not consult capabilities; confused-deputy escalation impossible.
- **GRKERNSEC_HIDESYM symbol** — `sys_sysctl` is not a distinct kallsyms entry; resolves to `sys_ni_syscall`.
- **Audit ring-buffer flood-immune** — per-uid summary record at 1 Hz, not per-call.
- **Compat path equally stubbed** — compat (32-bit) callers cannot resurrect the interface via the compat table.
- **Kexec preservation** — legacy-slot binding rebuilt identically on post-kexec kernel.

