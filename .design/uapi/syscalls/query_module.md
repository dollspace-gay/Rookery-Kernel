# Tier-5 syscall: query_module(2) — syscall 178 (DEPRECATED)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys_ni.c (COND_SYSCALL(query_module))
  - arch/x86/entry/syscalls/syscall_64.tbl (178  64  query_module)
  - Documentation/ABI/removed/query_module (historical)
  - /sys/module/* (modern replacement)
-->

## Summary

`query_module(2)` is a **deprecated, permanently-stubbed** syscall whose original Linux 2.0 / 2.2 / 2.4 role was to inspect loaded kernel modules: query module list, per-module info (size, refcount, flags), exported symbols, and module dependencies. The `lsmod` / `modinfo` tools of that era were thin wrappers over this syscall. In Linux 2.6 (2003) the entire interface was replaced by the `/sys/module/` sysfs hierarchy, where each loaded module exposes a directory with `refcnt`, `coresize`, `holders/`, `sections/`, `parameters/`, `notes/`, and `taint` attributes. Modern `kmod` (the umbrella for `lsmod`, `modinfo`, `modprobe`, `depmod`) reads sysfs and `/proc/modules`; nothing uses syscall 178.

Today the slot returns `-ENOSYS` unconditionally. The syscall is documented for ABI-stability, security-audit signatures (a process calling syscall 178 is reconnaissance for legacy module-state disclosure), and grsec deprecated-syscall policy.

Critical for: ABI-table completeness, intrusion-detection (rootkit module-discovery probes), grsec deprecated-syscall audit, defence against module-info downgrade probing.

## Signature

```c
/* Pre-2.6 historical prototype, retained ONLY for headers */
int query_module(const char *name, int which, void *buf, size_t bufsize, size_t *ret);
```

```c
/* Modern (Linux 2.6+) effective prototype: stub */
long query_module(const char __user *name, int which,
                  void __user *buf, size_t bufsize,
                  size_t __user *ret);  /* always -ENOSYS */
```

```c
/* Historical 'which' values, preserved for reference */
#define QM_MODULES   1    /* list of loaded modules */
#define QM_DEPS      2    /* per-module dependencies */
#define QM_REFS      3    /* per-module references */
#define QM_SYMBOLS   4    /* per-module exported symbols */
#define QM_INFO      5    /* per-module struct module_info */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `name` | `const char __user *` | in (ignored) | Historical: target module name, or NULL for QM_MODULES. |
| `which` | `int` | in (ignored) | Historical: QM_MODULES / QM_DEPS / QM_REFS / QM_SYMBOLS / QM_INFO. |
| `buf` | `void __user *` | in/out (ignored) | Historical: output buffer. |
| `bufsize` | `size_t` | in (ignored) | Historical: buffer size in bytes. |
| `ret` | `size_t __user *` | out (ignored) | Historical: bytes (or entries) written. |

All arguments are unread by the modern stub.

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
__NR_query_module (x86_64)    = 178    /* slot reserved, stubbed */
__NR_query_module (i386)      = 167    /* slot reserved, stubbed */
__NR_query_module (arm64)     = N/A    /* never assigned */
__NR_query_module (riscv)     = N/A
__NR_query_module (loongarch) = N/A

/* sys_ni.c:
 *     COND_SYSCALL(query_module);
 */
```

## Compatibility contract

REQ-1: Syscall number is **178** on x86_64; **167** on i386. Numbers reserved-forever.

REQ-2: Every invocation MUST return `-ENOSYS`. No argument validation, no `copy_from_user`, no `copy_to_user`, no LSM hook from the stub itself.

REQ-3: The stub MUST NOT enumerate `struct module` list, MUST NOT touch `module_mutex`, MUST NOT call `find_module`, MUST NOT walk symbol tables.

REQ-4: Historical `which` semantics MUST NOT be honoured even partially. A stricter mode (grsec) may convert the call into `SIGSYS` instead of `-ENOSYS`.

REQ-5: On architectures that never assigned the number, `-ENOSYS` originates at the syscall-table boundary.

REQ-6: Modern userspace MUST use `/sys/module/<name>/` and `/proc/modules` for module-state introspection. These are subject to `CAP_SYSLOG` / `kptr_restrict` for address fields.

REQ-7: `seccomp` filters MAY allowlist or denylist syscall 178 explicitly.

REQ-8: `ptrace` of the stub sees syscall-entry/exit pair with return `-ENOSYS`.

REQ-9: `sys_query_module` is NOT a distinct kallsyms entry; the slot resolves to `__x64_sys_ni_syscall`.

REQ-10: ABI-stability: slot 178 MUST NOT be reassigned.

REQ-11: Per-Rookery: legacy slot reserved at compile time; dispatches to shared `sys_ni::enosys()`.

## Acceptance Criteria

- [ ] AC-1: `syscall(SYS_query_module, NULL, QM_MODULES, NULL, 0, NULL)` returns `-1, ENOSYS`.
- [ ] AC-2: `syscall(178, "foo", QM_INFO, buf, sizeof buf, &ret)` returns `-1, ENOSYS`; buf and ret unmodified.
- [ ] AC-3: Stub returns `-ENOSYS` regardless of whether modules subsystem is initialised.
- [ ] AC-4: Stub returns `-ENOSYS` regardless of `CAP_SYS_MODULE` / `CAP_SYSLOG`.
- [ ] AC-5: `seccomp` filter denying `query_module` produces configured action.
- [ ] AC-6: `strace -e query_module` traces one `query_module(...) = -1 ENOSYS`.
- [ ] AC-7: Architectures without the slot return `-ENOSYS` via the table boundary.
- [ ] AC-8: Grsec-enabled kernel logs each invocation under `GRKERNSEC_DEPRECATED_SYSCALL_AUDIT`.
- [ ] AC-9: Boot-time sys_call_table integrity check confirms slot 178 = `sys_ni_syscall`.
- [ ] AC-10: 1e6 calls under stress allocate no kernel memory and acquire no lock.
- [ ] AC-11: `/sys/module/*` remains the canonical interface and is unaffected.
- [ ] AC-12: Calls under each of which ∈ {QM_MODULES, QM_DEPS, QM_REFS, QM_SYMBOLS, QM_INFO} return `-ENOSYS` identically.

## Architecture

```rust
#[syscall(nr = 178, abi = "sysv", legacy = true)]
pub fn sys_query_module(
    _name: UserPtr<u8>,
    _which: i32,
    _buf: UserPtr<u8>,
    _bufsize: usize,
    _ret: UserPtr<usize>,
) -> isize {
    SysNi::enosys("query_module")
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
| `no_module_list_walk` | INVARIANT | per-stub: no traversal of `modules` global list. |
| `no_module_mutex` | INVARIANT | per-stub: `module_mutex` not acquired. |
| `no_user_copy_in_or_out` | INVARIANT | per-stub: no copy_from_user / copy_to_user. |
| `no_cap_check` | INVARIANT | per-stub: no `capable(...)` call. |
| `no_alloc` | INVARIANT | per-stub: no kmalloc / vmalloc. |

### Layer 2: TLA+

`kernel/legacy-syscalls.tla`:
- States: per-syscall-table slot binding, per-stub invocation.
- Properties:
  - `safety_slot_178_bound_to_ni` — per-boot: slot 178 → sys_ni_syscall.
  - `safety_stub_returns_enosys` — per-call: ret == -ENOSYS.
  - `safety_no_module_disclosure` — per-call: no `struct module` field observable to userspace.
  - `liveness_terminates` — per-call: O(1) steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_query_module` post: ret == -ENOSYS | `sys_query_module` |
| `sys_query_module` post: no observable state mutation (except audit log) | `sys_query_module` |
| `sys_query_module` post: no `struct module` field referenced | `sys_query_module` |

### Layer 4: Verus / Creusot functional

Per-`query_module(2)` man-page ("This system call is present only in kernels before Linux 2.6.") equivalence with `sys_ni_syscall`. `strace` output equivalence verified.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`query_module(2)` reinforcement:

- **Per-stub no-module-state-disclosure** — defense against per-info-leak (refcounts, addresses, symbol tables).
- **Per-stub no-user-write** — defense against per-userspace-corruption via stale buffer pointer.
- **Per-stub no-module-mutex** — defense against per-stub-induced lock contention.
- **Per-stub no-alloc** — defense against per-DoS via legacy-syscall flood.
- **Per-stub constant-time** — defense against per-side-channel timing probe.
- **Per-syscall-table-integrity** — boot-time + periodic verification slot 178 → sys_ni_syscall.
- **Per-seccomp filterable** — userspace can block the slot.
- **Per-sysfs gating intact** — `/sys/module/<name>/` remains the only path; gated by usual VFS permission checks.

## Grsecurity / PaX-style Reinforcement

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every invocation logged with `pid`, `uid`, `comm`, `exe path`, `which` value (still captured from the unread arg via syscall-entry tracepoint), timestamp. Rate-limited per-uid. Very high signal: `which=QM_SYMBOLS` is a classic rootkit/symbol-discovery probe.
- **Boot-time sys_call_table verification** — slot 178 asserted `sys_ni_syscall`; mismatch panics.
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family** — optional stricter mode: invocation triggers `SIGSYS` to the caller.
- **GRKERNSEC_MODHARDEN coverage** — module-state introspection is gated to `CAP_SYS_MODULE` in hardened mode (the modern path); the legacy stub cannot bypass.
- **GRKERNSEC_HIDESYM** — module addresses in `/sys/module/<name>/sections/*` are hidden from non-root regardless of `kptr_restrict`; even a buggy reintroduction of #178 would be guarded by HIDESYM-style filtering at every emission point.
- **PaX KERNEXEC on syscall-table page** — slot-hook attempts fault.
- **PAX_RANDKSTACK at stub entry** — randomizes kernel-stack offset per call.
- **No CAP_SYSLOG elevation possible** — stub does not consult capabilities; confused-deputy escalation impossible.
- **GRKERNSEC_HIDESYM symbol** — `sys_query_module` is not a distinct kallsyms entry.
- **Audit ring-buffer flood-immune** — per-uid summary record at 1 Hz, not per-call.
- **Kexec preservation** — the legacy-slot binding rebuilt identically on post-kexec kernel.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `/sys/module/*` sysfs interface (Tier-3 `kernel/module/sysfs.md`).
- `/proc/modules` (Tier-3 `fs/proc/modules.md`).
- `init_module(2)` / `finit_module(2)` / `delete_module(2)` (Tier-5 separate docs).
- `sys_ni_syscall` shared stub (Tier-3 `kernel/sys_ni.md`).
- Grsec audit subsystem (Tier-3 `security/grsec/audit.md`).
- Implementation code.
