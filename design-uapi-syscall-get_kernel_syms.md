---
title: "Tier-5 syscall: get_kernel_syms(2) — syscall 177 (DEPRECATED)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`get_kernel_syms(2)` is a **deprecated, permanently-stubbed** syscall whose original Linux 0.99 / 1.x / 2.0 role was to copy the kernel's exported-symbol table to userspace so that the pre-1996 `insmod` could perform module relocation entirely in userspace. The old workflow was: (1) userspace called `get_kernel_syms(NULL)` to learn the size, (2) called `get_kernel_syms(buf)` to read the table, (3) relocated the module against the returned symbols, (4) used `create_module(2)` + `init_module(2)` to load the relocated text. This entire pipeline was retired in Linux 2.6 (2003) when relocation moved into the kernel and the loader switched to `init_module(2)`'s self-contained ELF.

Today the slot returns `-ENOSYS` unconditionally. `/proc/kallsyms` is the canonical modern interface for kernel-symbol enumeration (and is heavily restricted via `kptr_restrict`). The syscall is documented for ABI-stability, security-audit signatures (a process calling syscall 177 on a modern kernel is reconnaissance), and grsec deprecated-syscall policy.

Critical for: ABI-table completeness, intrusion-detection (rootkit symbol-discovery probes), grsec deprecated-syscall audit, defence against module-loader downgrade probing.

### Acceptance Criteria

- [ ] AC-1: `syscall(SYS_get_kernel_syms, NULL)` on x86_64 returns `-1` with `errno == ENOSYS`.
- [ ] AC-2: `syscall(177, buf)` with valid buf returns `-1, ENOSYS`; buf is NOT written.
- [ ] AC-3: `syscall(177, ...)` as root with `CAP_SYSLOG` returns `-ENOSYS`.
- [ ] AC-4: `kptr_restrict = 0` does not change the stub return.
- [ ] AC-5: `seccomp` filter denying `get_kernel_syms` produces the configured action.
- [ ] AC-6: `strace -e get_kernel_syms` traces one `get_kernel_syms(...) = -1 ENOSYS`.
- [ ] AC-7: Architectures without the slot return `-ENOSYS` via the table boundary.
- [ ] AC-8: Grsec-enabled kernel logs each invocation under `GRKERNSEC_DEPRECATED_SYSCALL_AUDIT`.
- [ ] AC-9: Boot-time sys_call_table integrity check confirms slot 177 = `sys_ni_syscall`.
- [ ] AC-10: 1e6 calls under stress allocate no kernel memory and take no lock.
- [ ] AC-11: `/proc/kallsyms` remains the canonical interface and is unaffected by stub invocations.
- [ ] AC-12: Calls from non-init userns and from chroots all return `-ENOSYS` identically.

### Architecture

```rust
#[syscall(nr = 177, abi = "sysv", legacy = true)]
pub fn sys_get_kernel_syms(_table: UserPtr<KernelSym>) -> isize {
    SysNi::enosys("get_kernel_syms")
}
```

`SysNi::enosys(name: &'static str) -> isize`:
1. /* Shared no-op return path for every stubbed legacy syscall */
2. #[cfg(feature = "grsec_deprecated_audit")]
3.   GrsecAudit::log_legacy_syscall(name, current_creds());
4. -ENOSYS

`GrsecAudit::log_legacy_syscall(name, creds)` (shared with all legacy stubs):
1. let task = current();
2. let event = LegacySyscallEvent {
3.   name,
4.   pid: task.tgid_in(init_pid_ns()),
5.   uid: creds.euid,
6.   exe: task.exe_file_path(),
7.   timestamp: now_monotonic_ns(),
8. };
9. AuditQueue::enqueue(event);

`SysCallTable::verify_legacy_slots()` boot-time check:
1. for (nr, expected) in LEGACY_STUB_SLOTS {
2.   if sys_call_table[nr] != sys_ni_syscall as *const _ {
3.     panic!("syscall table corruption at slot {}", nr);
4.   }
5. }

### Out of Scope

- `/proc/kallsyms` (Tier-3 `kernel/kallsyms.md`).
- `kptr_restrict` sysctl semantics (Tier-3 `kernel/sysctl-kptr.md`).
- `init_module(2)` / `finit_module(2)` (Tier-5 separate docs).
- `sys_ni_syscall` shared stub (Tier-3 `kernel/sys_ni.md`).
- Grsec audit subsystem (Tier-3 `security/grsec/audit.md`).
- Implementation code.

### signature

```c
/* Pre-2.6 historical prototype, retained ONLY for headers */
int get_kernel_syms(struct kernel_sym *table);
```

```c
/* Modern (Linux 2.6+) effective prototype: stub */
long get_kernel_syms(struct kernel_sym __user *table);  /* always -ENOSYS */
```

```c
/* Historical structure layout, preserved for reference only */
struct kernel_sym {
    unsigned long value;        /* kernel address */
    char          name[60];     /* symbol name, NUL-padded */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `table` | `struct kernel_sym __user *` | in (ignored) | Historical: NULL to query count, or buffer for output. Modern stub: unread. |

The modern stub does not dereference the pointer.

### return value

| Value | Meaning |
|---|---|
| `-1` + `errno = ENOSYS` | Always. |

There is no success path. Historical semantics (return entry count) are gone.

### errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always. Returned unconditionally from `sys_ni_syscall`. |

### abi surface

```text
__NR_get_kernel_syms (x86_64)    = 177    /* slot reserved, stubbed */
__NR_get_kernel_syms (i386)      = 130    /* slot reserved, stubbed */
__NR_get_kernel_syms (arm64)     = N/A    /* never assigned */
__NR_get_kernel_syms (riscv)     = N/A
__NR_get_kernel_syms (loongarch) = N/A

/* Architectures using generic-syscall (asm-generic/unistd.h) never assigned
 * a number to this legacy interface; the slot is x86-legacy only.
 */

/* sys_ni.c:
 *     COND_SYSCALL(get_kernel_syms);
 * compiles to a weak alias of sys_ni_syscall returning -ENOSYS.
 */
```

### compatibility contract

REQ-1: Syscall number is **177** on x86_64; **130** on i386. The numbers are reserved-forever.

REQ-2: Every invocation MUST return `-ENOSYS`. No argument validation, no `copy_to_user`, no LSM hook from the stub path itself (syscall-entry tracepoint still fires).

REQ-3: The stub MUST NOT consult `current_cred()`, `kallsyms_lookup_*`, `kptr_restrict`, or any capability. It returns `-ENOSYS` even for root.

REQ-4: The stub MUST NOT honour the historical "query-count via NULL pointer" semantics. There is no count to return.

REQ-5: On architectures that never assigned the number, `-ENOSYS` originates from the syscall-table boundary, not the stub.

REQ-6: Modern userspace MUST use `/proc/kallsyms` (subject to `kptr_restrict` and `CAP_SYSLOG`) for kernel-symbol enumeration. The stub does NOT provide a fallback path.

REQ-7: `seccomp` filters MAY allowlist or denylist syscall 177 explicitly.

REQ-8: `ptrace` of the stub sees syscall-entry/exit pair with return `-ENOSYS`.

REQ-9: `sys_get_kernel_syms` is NOT exported as a distinct kallsyms entry; the slot resolves to `__x64_sys_ni_syscall`.

REQ-10: ABI-stability: slot 177 MUST NOT be reassigned to any new functionality.

REQ-11: Per-Rookery: legacy slot reserved at compile time; dispatches to shared `sys_ni::enosys()` with optional grsec audit hook.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_enosys` | INVARIANT | per-call: return value is exactly `-ENOSYS`. |
| `no_kallsyms_lookup` | INVARIANT | per-stub: no `kallsyms_lookup_*` invocation. |
| `no_user_copy` | INVARIANT | per-stub: no `copy_to_user`. |
| `no_cap_check` | INVARIANT | per-stub: no `capable(...)` call. |
| `no_alloc` | INVARIANT | per-stub: no kmalloc / vmalloc. |
| `no_lock_taken` | INVARIANT | per-stub: no kernel lock acquired. |

### Layer 2: TLA+

`kernel/legacy-syscalls.tla` (shared):
- States: per-syscall-table slot binding, per-stub invocation, per-audit-queue enqueue.
- Properties:
  - `safety_slot_177_bound_to_ni` — per-boot: slot 177 → sys_ni_syscall.
  - `safety_stub_returns_enosys` — per-call: ret == -ENOSYS.
  - `safety_no_symbol_disclosure` — per-call: no kernel-symbol bytes copied to userspace.
  - `liveness_terminates` — per-call: O(1) steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_get_kernel_syms` post: ret == -ENOSYS | `sys_get_kernel_syms` |
| `sys_get_kernel_syms` post: no observable state mutation (except audit log) | `sys_get_kernel_syms` |
| `sys_get_kernel_syms` post: no kernel address materialized at any userspace-reachable location | `sys_get_kernel_syms` |

### Layer 4: Verus / Creusot functional

Per-`get_kernel_syms(2)` man-page (NOTES: "This system call is present only in kernels before Linux 2.6.") equivalence with `sys_ni_syscall`. `strace` output equivalence verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`get_kernel_syms(2)` reinforcement:

- **Per-stub no-symbol-table-access** — defense against per-info-leak; the implementation cannot accidentally regress to exposing symbols.
- **Per-stub no-user-write** — defense against per-userspace-corruption via stale buffer pointer.
- **Per-stub no-alloc** — defense against per-DoS via legacy-syscall flood.
- **Per-stub constant-time** — defense against per-side-channel timing probe.
- **Per-syscall-table-integrity** — boot-time + periodic verification slot 177 → sys_ni_syscall.
- **Per-seccomp filterable** — userspace can block the slot without kernel-side carve-out.
- **Per-kallsyms gating intact** — `/proc/kallsyms` remains the only path; subject to `kptr_restrict` and `CAP_SYSLOG`.

### grsecurity / pax-style reinforcement

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every invocation logged with `pid`, `uid`, `comm`, `exe path`, timestamp. Rate-limited per-uid. Particularly important for syscall 177 because it is a high-signal reconnaissance indicator: legitimate software hasn't called this in two decades.
- **Boot-time sys_call_table verification** — slot 177 asserted `sys_ni_syscall`; mismatch panics. Rootkits that historically hooked this slot to silently service kernel-symbol disclosure are blocked at boot.
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family** — optional stricter mode: invocation of any deprecated-stub syscall (174, 177, 178, 180, 156, 236, 183, 182) triggers `SIGSYS` to the caller. Enabled in hardened distributions where no legitimate process needs these slots.
- **GRKERNSEC_HIDESYM coverage** — even if a future buggy patch reintroduced symbol-table copying, HIDESYM would suppress kernel-pointer leakage in audit messages, logs, and any user-observable path.
- **Kernel-symbol enumeration funneled through `/proc/kallsyms`** — which is itself gated on `kptr_restrict ≥ 1` (zero kernel pointers for non-root) or ≥ 2 (zero for everyone). The legacy slot cannot bypass this gating.
- **PaX KERNEXEC on syscall-table page** — `sys_call_table` mapped read-only after init; rootkit hook attempts fault.
- **PAX_RANDKSTACK at stub entry** — randomizes kernel-stack offset per call; defeats stack-layout fingerprinting via #177 probes.
- **No CAP_SYSLOG elevation possible** — the stub is deliberately written to NOT consult capabilities; confused-deputy escalation impossible.
- **GRKERNSEC_HIDESYM symbol** — `sys_get_kernel_syms` does not exist as a distinct kallsyms entry.
- **Audit ring-buffer flood-immune** — per-uid summary record at 1 Hz, not per-call.
- **Kexec preservation** — the legacy-slot binding rebuilt identically on post-kexec kernel.

